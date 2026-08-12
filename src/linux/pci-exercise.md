---
minutes: 50
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Exercise: QEMU EDU PCI DRM Driver

In this exercise, you will write a real PCI device driver for the **QEMU `edu` educational device** using the **DRM (Direct Rendering Manager) subsystem** class interface.

The driver will map the device's Base Address Register (BAR 0) to access hardware registers, register an interrupt handler to handle MSI interrupts, and expose a `/dev/dri/cardX` interface to compute factorials and test card liveness.

---

## Booting QEMU with the EDU Device

To run the VM with the virtual PCI card inserted, append the `-device edu` option to the QEMU command:

```bash
cd ~/learn-rust
qemu-system-x86_64 \
    -machine q35,acpi=on \
    -kernel linux/arch/x86/boot/bzImage \
    -drive file=debian.img,format=raw,if=virtio \
    -append "root=/dev/vda console=ttyS0 acpi=force" \
    -nographic \
    -no-reboot \
    -m 2G -smp 2 \
    -virtfs local,path=$PWD,mount_tag=hostshare,security_model=none,id=hostshare \
    -device edu
```

Once booted, verify that the device is detected on the PCIe bus:

```bash
lspci -d 1234:11e8 -v
```

---

## Hardware Register Map

The device registers are located at the following offsets inside **BAR 0** (size `0x80` bytes):

| Register | Offset | Access | Width | Description |
| :--- | :---: | :---: | :---: | :--- |
| **`ID`** | `0x00` | RO | 32-bit | Returns identification register: `0x010000ed`. |
| **`LIVENESS`** | `0x04` | RW | 32-bit | Writing value `X` returns `~X` when read back. |
| **`FACTORIAL`** | `0x08` | RW | 32-bit | Write `N` to start computing `N!`. Read it back to get the result. |
| **`STATUS`** | `0x20` | RO | 32-bit | Bit 0: `1` if computing factorial, `0` if idle.<br>Bit 7: `1` if an interrupt is raised. |
| **`IRQ_STATUS`** | `0x24` | RW | 32-bit | Read pending interrupt value. Write the same value back to clear/acknowledge it. |
| **`IRQ_RAISE`** | `0x60` | WO | 32-bit | Write any value here to raise an interrupt (MSI) for testing. |

---

## Stub Code

Below is the starter code for your driver. Copy this into a new source file (e.g., `samples/rust/rust_driver_pci_edu_drm.rs`). Your task is to implement the missing register access and interrupt handling parts (marked `// TODO`).

```rust,ignore
// SPDX-License-Identifier: GPL-2.0
//! Rust PCI EDU driver sample with a DRM class device interface and IRQ support.

use kernel::{
    device,
    device::{Bound, Core, DeviceContext},
    drm,
    drm::ioctl,
    drm::Registered,
    io::{poll, register, Io},
    irq,
    pci,
    prelude::*,
    sync::aref::ARef,
    time,
    uapi,
};

mod regs {
    use super::*;
    register! {
        pub(super) ID(u32) @ 0x00 {
            31:0 id;
        }
        pub(super) LIVENESS(u32) @ 0x04 {
            31:0 val;
        }
        pub(super) FACTORIAL(u32) @ 0x08 {
            31:0 val;
        }
        pub(super) STATUS(u32) @ 0x20 {
            0:0 computing;
            7:7 irq;
        }
        pub(super) IRQ_STATUS(u32) @ 0x24 {
            31:0 val;
        }
        pub(super) IRQ_RAISE(u32) @ 0x60 {
            31:0 val;
        }
    }
    pub(super) const END: usize = 0x80;
}

#[pin_data]
struct EduRegistrationData<'drm> {
    pdev: &'drm pci::Device<Bound>,
    bar: pci::Bar<'drm, { regs::END }>,
}

#[pin_data]
struct EduIrqHandler {
    drm: ARef<drm::Device<EduDrmDriver>>,
}

impl irq::Handler for EduIrqHandler {
    fn handle(&self, _device: &device::Device<Bound>) -> irq::IrqReturn {
        let guard = match self.drm.registration_guard() {
            Some(guard) => guard,
            None => return irq::IrqReturn::None,
        };

        guard.registration_data_with(|reg_data| {
            // TODO: Read IRQ_STATUS register. If it is 0, return IrqReturn::None.
            // TODO: Log the interrupt using dev_info! on reg_data.pdev.
            // TODO: Write status back to IRQ_STATUS to clear/acknowledge the interrupt.
            irq::IrqReturn::Handled
        })
    }
}

struct EduFile;

impl drm::file::DriverFile for EduFile {
    type Driver = EduDrmDriver;

    fn open(_dev: &drm::Device<EduDrmDriver>) -> Result<Pin<KBox<Self>>> {
        Ok(KBox::new(Self, GFP_KERNEL)?.into())
    }
}

impl EduFile {
    pub(crate) fn get_id(
        _dev: &drm::Device<EduDrmDriver, Registered>,
        reg_data: &EduRegistrationData<'_>,
        arg: &mut uapi::drm_edu_get_id,
        _file: &drm::File<Self>,
    ) -> Result<u32> {
        // TODO: Read regs::ID and write it to arg.id
        Ok(0)
    }

    pub(crate) fn test_liveness(
        _dev: &drm::Device<EduDrmDriver, Registered>,
        reg_data: &EduRegistrationData<'_>,
        arg: &mut uapi::drm_edu_test_liveness,
        _file: &drm::File<Self>,
    ) -> Result<u32> {
        // TODO: Write arg.val to regs::LIVENESS
        // TODO: Read regs::LIVENESS and write it to arg.inv
        Ok(0)
    }

    pub(crate) fn compute_factorial(
        _dev: &drm::Device<EduDrmDriver, Registered>,
        reg_data: &EduRegistrationData<'_>,
        arg: &mut uapi::drm_edu_compute_factorial,
        _file: &drm::File<Self>,
    ) -> Result<u32> {
        // TODO: Write arg.val to regs::FACTORIAL
        // TODO: Poll STATUS.computing until it is 0 using read_poll_timeout
        // TODO: Read result from regs::FACTORIAL and write it to arg.res
        Ok(0)
    }

    pub(crate) fn test_irq(
        _dev: &drm::Device<EduDrmDriver, Registered>,
        reg_data: &EduRegistrationData<'_>,
        arg: &mut uapi::drm_edu_test_irq,
        _file: &drm::File<Self>,
    ) -> Result<u32> {
        // TODO: Write arg.val to regs::IRQ_RAISE to trigger interrupt
        Ok(0)
    }
}

#[pin_data]
struct EduObject {}

impl drm::gem::DriverObject for EduObject {
    type Driver = EduDrmDriver;
    type Args = ();

    fn new(
        _dev: &drm::Device<EduDrmDriver>,
        _size: usize,
        _args: Self::Args,
    ) -> impl PinInit<Self, Error> {
        try_pin_init!(EduObject {})
    }
}

struct EduDrmDriver;

#[vtable]
impl drm::Driver for EduDrmDriver {
    type Data = ();
    type RegistrationData<'drm> = EduRegistrationData<'drm>;
    type File = EduFile;
    type Object = drm::gem::Object<EduObject>;
    type ParentDevice<Ctx: DeviceContext> = pci::Device<Ctx>;

    const INFO: drm::DriverInfo = drm::DriverInfo {
        major: 0,
        minor: 0,
        patchlevel: 0,
        name: c"qemu-edu-drm",
        desc: c"QEMU PCI EDU DRM Driver",
    };

    const FEAT_RENDER: bool = true;

    kernel::declare_drm_ioctls! {
        (EDU_GET_ID, drm_edu_get_id, ioctl::RENDER_ALLOW, EduFile::get_id),
        (EDU_TEST_LIVENESS, drm_edu_test_liveness, ioctl::RENDER_ALLOW, EduFile::test_liveness),
        (EDU_COMPUTE_FACTORIAL, drm_edu_compute_factorial, ioctl::RENDER_ALLOW, EduFile::compute_factorial),
        (EDU_TEST_IRQ, drm_edu_test_irq, ioctl::RENDER_ALLOW, EduFile::test_irq),
    }
}

#[pin_data(PinnedDrop)]
struct EduDriverData<'bound> {
    pdev: ARef<pci::Device>,
    #[pin]
    _irq: irq::Registration<EduIrqHandler>,
    _reg: drm::Registration<'bound, EduDrmDriver>,
}

struct EduDriver;

kernel::pci_device_table!(
    PCI_TABLE,
    <EduDriver as pci::Driver>::IdInfo,
    [(
        pci::DeviceId::from_id(pci::Vendor::QEMU, 0x11e8),
        ()
    )]
);

impl pci::Driver for EduDriver {
    type IdInfo = ();
    type Data<'bound> = EduDriverData<'bound>;

    const ID_TABLE: pci::IdTable<Self::IdInfo> = &PCI_TABLE;

    fn probe<'bound>(
        pdev: &'bound pci::Device<Core<'_>>,
        _info: Option<&'bound Self::IdInfo>,
    ) -> impl PinInit<Self::Data<'bound>, Error> + 'bound {
        pin_init::pin_init_scope(move || {
            pdev.enable_device_mem()?;
            pdev.set_master();

            let bar = pdev.iomap_region_sized::<{ regs::END }>(0, c"qemu_edu_drm")?;

            let unreg_dev = drm::UnregisteredDevice::<EduDrmDriver>::new(pdev, Ok(()))?;

            let reg_data = pin_init!(EduRegistrationData {
                pdev,
                bar,
            });

            let _reg = unsafe {
                drm::Registration::new(pdev.as_ref(), unreg_dev, reg_data, 0)?
            };

            let drm_ref: ARef<drm::Device<EduDrmDriver>> = _reg.device().into();

            let vectors = pdev.alloc_irq_vectors(1, 1, pci::IrqTypes::all())?;
            let vector = *vectors.start();

            let irq_init = pdev.request_irq(
                vector,
                irq::Flags::SHARED,
                c"qemu_edu_drm",
                try_pin_init!(EduIrqHandler {
                    drm: drm_ref,
                }),
            );

            Ok(try_pin_init!(EduDriverData {
                pdev: pdev.into(),
                _irq <- irq_init,
                _reg,
            }))
        })
    }
}

#[pinned_drop]
impl PinnedDrop for EduDriverData<'_> {
    fn drop(self: Pin<&mut Self>) {
        dev_info!(self.pdev, "Remove QEMU EDU PCI DRM driver.\n");
    }
}

kernel::module_pci_driver! {
    type: EduDriver,
    name: "rust_driver_pci_edu_drm",
    authors: ["Alice Ryhl"],
    description: "QEMU PCI EDU DRM driver",
    license: "GPL v2",
}
```
