---
minutes: 15
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Exposing Class Devices

A PCI driver operates hardware, but it must expose a **class device** (like a character device or DRM device) so userspace applications can interact with it.

---

## Option 1: Exposing via `miscdevice`

A simple character device interface (`/dev/qemu-edu-misc`) can be added by registering a `MiscDeviceRegistration` within our PCI driver state.

```rust,ignore
// SPDX-License-Identifier: GPL-2.0
//! PCI driver exposing a miscdevice.

use kernel::{
    device::Core,
    fs::File,
    miscdevice::{MiscDevice, MiscDeviceOptions, MiscDeviceRegistration},
    pci,
    prelude::*,
};

struct EduDriver;

#[pin_data]
struct EduPciData {
    pdev: ARef<pci::Device>,
    #[pin]
    _miscdev: MiscDeviceRegistration<EduMiscDevice>,
}

struct EduMiscDevice {}

#[vtable]
impl MiscDevice for EduMiscDevice {
    type Data = ();
    type Ptr = Pin<KBox<Self>>;

    fn open(_file: &File, _misc: &MiscDeviceRegistration<Self>) -> Result<Pin<KBox<Self>>> {
        KBox::try_pin_init(try_pin_init!(EduMiscDevice {}), GFP_KERNEL)
    }
}

kernel::pci_device_table!(
    PCI_TABLE,
    <EduDriver as pci::Driver>::IdInfo,
    [(pci::DeviceId::from_id(pci::Vendor::QEMU, 0x11e8), ())]
);

impl pci::Driver for EduDriver {
    type IdInfo = ();
    type Data<'bound> = EduPciData;

    const ID_TABLE: pci::IdTable<Self::IdInfo> = &PCI_TABLE;

    fn probe<'bound>(
        pdev: &'bound pci::Device<Core<'_>>,
        _info: Option<&'bound Self::IdInfo>,
    ) -> impl PinInit<Self::Data<'bound>, Error> + 'bound {
        pin_init::pin_init_scope(move || {
            dev_info!(pdev, "Probing QEMU EDU PCI misc device!\n");
            
            pdev.enable_device_mem()?;
            pdev.set_master();

            let options = MiscDeviceOptions {
                name: c"qemu-edu-misc",
                parent: Some(pdev.as_ref()),
            };

            let miscdev_init = MiscDeviceRegistration::register(options, ());

            Ok(try_pin_init!(EduPciData {
                pdev: pdev.into(),
                _miscdev <- miscdev_init,
            }))
        })
    }
}

kernel::module_pci_driver! {
    type: EduDriver,
    name: "rust_driver_pci_misc",
    authors: ["Your Name"],
    description: "QEMU PCI EDU driver with miscdevice",
    license: "GPL v2",
}
```

---

## Option 2: Exposing via the DRM Subsystem

For graphics cards and accelerators, the **Direct Rendering Manager (DRM)** subsystem is preferred. A DRM device (`/dev/dri/cardX`) is registered using the `drm::Driver` trait.

*   **Single Driver Type:** The same type `EduDriver` can implement both `pci::Driver` and `drm::Driver` traits.
*   **SRCU Protection:** DRM uses a sleepable SRCU critical section (`drm::RegistrationGuard`) to guarantee memory safety during IOCTLs.
*   **Registration Data:** Safe access to the BAR registers is passed via `EduDrmData` to the file operations.
*   **GEM Objects:** `drm::gem::Object` handles GPU memory allocations.

```rust,ignore
// SPDX-License-Identifier: GPL-2.0
//! Minimal DRM PCI driver with GET_ID IOCTL (merged Driver types).

use kernel::{
    device::{Core, DeviceContext},
    drm,
    drm::ioctl,
    io::Io,
    pci,
    prelude::*,
    sync::aref::ARef,
    uapi,
};

#[allow(dead_code)]
const EDU_GET_ID: u32 = kernel::ioctl::_IOR::<u32>('E' as u32, 0x00);

mod regs {
    use kernel::io::register;
    register! {
        pub(super) ID(u32) @ 0x00 {
            31:0 id;
        }
    }
    pub(super) const END: usize = 0x80;
}

struct EduDriver;

#[pin_data]
struct EduPciData<'bound> {
    pdev: ARef<pci::Device>,
    _reg: drm::Registration<'bound, EduDriver>,
}

#[pin_data]
struct EduDrmData<'a> {
    bar: pci::Bar<'a, { regs::END }>,
}

struct EduFile;

#[pin_data]
struct EduObject {}

impl pci::Driver for EduDriver {
    type IdInfo = ();
    type Data<'bound> = EduPciData<'bound>;

    const ID_TABLE: pci::IdTable<Self::IdInfo> = &PCI_TABLE;

    fn probe<'bound>(
        pdev: &'bound pci::Device<Core<'_>>,
        _info: Option<&'bound Self::IdInfo>,
    ) -> impl PinInit<Self::Data<'bound>, Error> + 'bound {
        pin_init::pin_init_scope(move || {
            pdev.enable_device_mem()?;
            pdev.set_master();

            let bar = pdev.iomap_region_sized::<{ regs::END }>(0, c"qemu_edu_drm")?;

            let unreg_dev = drm::UnregisteredDevice::<EduDriver>::new(pdev, Ok(()))?;

            let reg_data = try_pin_init!(EduDrmData {
                bar,
            });

            let reg = unsafe {
                drm::Registration::new(pdev.as_ref(), unreg_dev, reg_data, 0)?
            };

            Ok(try_pin_init!(EduPciData {
                pdev: pdev.into(),
                _reg: reg,
            }))
        })
    }
}

#[vtable]
impl drm::Driver for EduDriver {
    type Data = ();
    type RegistrationData<'drm> = EduDrmData<'drm>;
    type File = EduFile;
    type Object = drm::gem::Object<EduObject>;
    type ParentDevice<Ctx: DeviceContext> = pci::Device<Ctx>;

    const INFO: drm::DriverInfo = drm::DriverInfo {
        major: 1,
        minor: 0,
        patchlevel: 0,
        name: c"qemu-edu-drm",
        desc: c"QEMU PCI EDU DRM Driver",
    };

    const FEAT_RENDER: bool = true;

    kernel::declare_drm_ioctls! {
        (EDU_GET_ID, drm_edu_get_id, ioctl::RENDER_ALLOW, EduFile::get_id),
    }
}

impl drm::file::DriverFile for EduFile {
    type Driver = EduDriver;

    fn open(_dev: &drm::Device<EduDriver>) -> Result<Pin<KBox<Self>>> {
        Ok(KBox::new(Self, GFP_KERNEL)?.into())
    }
}

impl EduFile {
    fn get_id(
        _dev: &drm::Device<EduDriver, drm::Registered>,
        reg_data: &EduDrmData<'_>,
        arg: &mut uapi::drm_edu_get_id,
        _file: &drm::File<Self>,
    ) -> Result<u32> {
        let id = reg_data.bar.read(regs::ID).id();
        arg.id = id.get();
        Ok(0)
    }
}

impl drm::gem::DriverObject for EduObject {
    type Driver = EduDriver;
    type Args = ();

    fn new(
        _dev: &drm::Device<EduDriver>,
        _size: usize,
        _args: Self::Args,
    ) -> impl PinInit<Self, Error> {
        try_pin_init!(EduObject {})
    }
}

kernel::pci_device_table!(
    PCI_TABLE,
    <EduDriver as pci::Driver>::IdInfo,
    [(pci::DeviceId::from_id(pci::Vendor::QEMU, 0x11e8), ())]
);

kernel::module_pci_driver! {
    type: EduDriver,
    name: "rust_driver_pci_drm",
    authors: ["Your Name"],
    description: "QEMU PCI EDU DRM driver",
    license: "GPL v2",
}
```

*   **Safe Unbind:** If the PCI card is unplugged or unbound, DRM prevents new IOCTL calls and safely revokes existing ones while the SRCU section finishes.

<details>

- Highlight the contrast: `miscdevice` is lightweight and simple, but has fewer built-in memory protection guarantees during dynamic unbind than the DRM subsystem's SRCU-guarded registry.
- Explain that `drm::gem::Object` handles GPU memory allocations, which are also tied into the DRM class device interface.

</details>
