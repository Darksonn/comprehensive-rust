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

A simple character device interface (`/dev/pci-testdev`) can be added by registering a `MiscDeviceRegistration` within our PCI driver state.

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

struct TestPciDriver;

#[pin_data]
struct TestPciData {
    pdev: ARef<pci::Device>,
    #[pin]
    _miscdev: MiscDeviceRegistration<TestPciMiscDevice>,
}

struct TestPciMiscDevice {}

#[vtable]
impl MiscDevice for TestPciMiscDevice {
    type Data = ();
    type Ptr = Pin<KBox<Self>>;

    fn open(_file: &File, _misc: &MiscDeviceRegistration<Self>) -> Result<Pin<KBox<Self>>> {
        KBox::try_pin_init(try_pin_init!(TestPciMiscDevice {}), GFP_KERNEL)
    }
}

kernel::pci_device_table!(
    PCI_TABLE,
    <TestPciDriver as pci::Driver>::IdInfo,
    [(pci::DeviceId::from_id(pci::Vendor::REDHAT, 0x5), ())]
);

impl pci::Driver for TestPciDriver {
    type IdInfo = ();
    type Data<'bound> = TestPciData;

    const ID_TABLE: pci::IdTable<Self::IdInfo> = &PCI_TABLE;

    fn probe<'bound>(
        pdev: &'bound pci::Device<Core<'_>>,
        _info: Option<&'bound Self::IdInfo>,
    ) -> impl PinInit<Self::Data<'bound>, Error> + 'bound {
        pin_init::pin_init_scope(move || {
            dev_info!(pdev, "Probing PCI testdev misc device!\n");
            
            pdev.enable_device_mem()?;
            pdev.set_master();

            let options = MiscDeviceOptions {
                name: c"pci-testdev",
                parent: Some(pdev.as_ref()),
            };

            let miscdev_init = MiscDeviceRegistration::register(options, ());

            Ok(try_pin_init!(TestPciData {
                pdev: pdev.into(),
                _miscdev <- miscdev_init,
            }))
        })
    }
}

kernel::module_pci_driver! {
    type: TestPciDriver,
    name: "rust_driver_pci_testdev_misc",
    authors: ["Your Name"],
    description: "PCI testdev misc driver",
    license: "GPL v2",
}
```

---

## Option 2: Exposing via the DRM Subsystem

For graphics cards and accelerators, the **Direct Rendering Manager (DRM)** subsystem is preferred. A DRM device (`/dev/dri/cardX`) is registered using the `drm::Driver` trait.

*   **Single Driver Type:** The same type `TestPciDriver` can implement both `pci::Driver` and `drm::Driver` traits.
*   **SRCU Protection:** DRM uses a sleepable SRCU critical section (`drm::RegistrationGuard`) to guarantee memory safety during IOCTLs.
*   **Minimal Registration:** At this stage, we register a bare DRM device without mapping BARs or exposing IOCTLs.

```rust,ignore
// SPDX-License-Identifier: GPL-2.0
//! Minimal DRM PCI driver (no MMIO or IOCTLs).

use kernel::{
    device::{Core, DeviceContext},
    drm,
    pci,
    prelude::*,
    sync::aref::ARef,
};

struct TestPciDriver;

#[pin_data]
struct TestPciData<'bound> {
    pdev: ARef<pci::Device>,
    _reg: drm::Registration<'bound, TestPciDriver>,
}

struct TestFile;

#[pin_data]
struct TestObject {}

impl pci::Driver for TestPciDriver {
    type IdInfo = ();
    type Data<'bound> = TestPciData<'bound>;

    const ID_TABLE: pci::IdTable<Self::IdInfo> = &PCI_TABLE;

    fn probe<'bound>(
        pdev: &'bound pci::Device<Core<'_>>,
        _info: Option<&'bound Self::IdInfo>,
    ) -> impl PinInit<Self::Data<'bound>, Error> + 'bound {
        pin_init::pin_init_scope(move || {
            pdev.enable_device_mem()?;
            pdev.set_master();

            let unreg_dev = drm::UnregisteredDevice::<TestPciDriver>::new(pdev, Ok(()))?;

            // We use () for RegistrationData as we don't share any data yet.
            let reg = unsafe {
                drm::Registration::new(pdev.as_ref(), unreg_dev, (), 0)?
            };

            Ok(try_pin_init!(TestPciData {
                pdev: pdev.into(),
                _reg: reg,
            }))
        })
    }
}

#[vtable]
impl drm::Driver for TestPciDriver {
    type Data = ();
    type RegistrationData<'drm> = ();
    type File = TestFile;
    type Object = drm::gem::Object<TestObject>;
    type ParentDevice<Ctx: DeviceContext> = pci::Device<Ctx>;

    const INFO: drm::DriverInfo = drm::DriverInfo {
        major: 1,
        minor: 0,
        patchlevel: 0,
        name: c"pci-testdev-drm",
        desc: c"PCI Testdev DRM Driver",
    };

    const FEAT_RENDER: bool = true;

    kernel::declare_drm_ioctls! {}
}

impl drm::file::DriverFile for TestFile {
    type Driver = TestPciDriver;

    fn open(_dev: &drm::Device<TestPciDriver>) -> Result<Pin<KBox<Self>>> {
        Ok(KBox::new(Self, GFP_KERNEL)?.into())
    }
}

impl drm::gem::DriverObject for TestObject {
    type Driver = TestPciDriver;
    type Args = ();

    fn new(
        _dev: &drm::Device<TestPciDriver>,
        _size: usize,
        _args: Self::Args,
    ) -> impl PinInit<Self, Error> {
        try_pin_init!(TestObject {})
    }
}

kernel::pci_device_table!(
    PCI_TABLE,
    <TestPciDriver as pci::Driver>::IdInfo,
    [(pci::DeviceId::from_id(pci::Vendor::REDHAT, 0x5), ())]
);

kernel::module_pci_driver! {
    type: TestPciDriver,
    name: "rust_driver_pci_testdev_drm",
    authors: ["Your Name"],
    description: "PCI Testdev DRM driver",
    license: "GPL v2",
}
```

*   **Safe Unbind:** If the PCI card is unplugged or unbound, DRM prevents new IOCTL calls and safely revokes existing ones while the SRCU section finishes.

<details>

- Highlight the contrast: `miscdevice` is lightweight and simple, but has fewer built-in memory protection guarantees during dynamic unbind than the DRM subsystem's SRCU-guarded registry.
- Explain that `drm::gem::Object` handles GPU memory allocations, which are also tied into the DRM class device interface.

</details>
