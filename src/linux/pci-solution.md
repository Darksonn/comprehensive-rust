---
minutes: 20
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Solution: QEMU EDU PCI DRM Driver

Here is the complete solution for the QEMU EDU PCI DRM driver, wrapping the hardware register mapping, MSI interrupts, and the DRM class device interface.

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
            let status = *reg_data.bar.read(regs::IRQ_STATUS).val();
            if status == 0 {
                return irq::IrqReturn::None;
            }

            dev_info!(
                reg_data.pdev,
                "QEMU EDU DRM IRQ handled! status=0x{:x}\n",
                status
            );
            
            // Clear interrupt status
            reg_data
                .bar
                .write_reg(regs::IRQ_STATUS::zeroed().with_val(status));

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
        arg.id = *reg_data.bar.read(regs::ID).id();
        Ok(0)
    }

    pub(crate) fn test_liveness(
        _dev: &drm::Device<EduDrmDriver, Registered>,
        reg_data: &EduRegistrationData<'_>,
        arg: &mut uapi::drm_edu_test_liveness,
        _file: &drm::File<Self>,
    ) -> Result<u32> {
        reg_data
            .bar
            .write_reg(regs::LIVENESS::zeroed().with_val(arg.val));
        arg.inv = *reg_data.bar.read(regs::LIVENESS).val();
        Ok(0)
    }

    pub(crate) fn compute_factorial(
        _dev: &drm::Device<EduDrmDriver, Registered>,
        reg_data: &EduRegistrationData<'_>,
        arg: &mut uapi::drm_edu_compute_factorial,
        _file: &drm::File<Self>,
    ) -> Result<u32> {
        reg_data
            .bar
            .write_reg(regs::FACTORIAL::zeroed().with_val(arg.val));

        // Sleep/poll until the card is finished:
        poll::read_poll_timeout(
            || Ok(reg_data.bar.read(regs::STATUS)),
            |status| status.computing() == 0,
            time::Delta::from_millis(1),
            time::Delta::from_millis(100),
        )?;

        arg.res = *reg_data.bar.read(regs::FACTORIAL).val();
        Ok(0)
    }

    pub(crate) fn test_irq(
        _dev: &drm::Device<EduDrmDriver, Registered>,
        reg_data: &EduRegistrationData<'_>,
        arg: &mut uapi::drm_edu_test_irq,
        _file: &drm::File<Self>,
    ) -> Result<u32> {
        reg_data
            .bar
            .write_reg(regs::IRQ_RAISE::zeroed().with_val(arg.val));
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

            let unreg_dev =
                drm::UnregisteredDevice::<EduDrmDriver>::new(pdev, Ok(()))?;

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
    description: "QEMU PCI EDU DRM Driver",
    license: "GPL v2",
}
```

<details>

- **SRCU Lifetime Guarantees:** Emphasize that the registration data is accessed via `guard.registration_data_with`, which wraps a DRM SRCU read lock. This protects against physical device hot-unplug while userspace is in the middle of executing a syscall.
- **Poll Timeout:** Note how `poll::read_poll_timeout` is used instead of active spinning. Since DRM runs in a process context (ioctl syscall), it can sleep to yield CPU cycles to other threads while waiting for the hardware to finish the calculation.

</details>
