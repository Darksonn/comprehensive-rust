---
minutes: 15
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Interrupt Handling

Devices signal asynchronous hardware events (such as DMA transfer completion) via hardware interrupts. In Rust, we handle these by implementing the **`irq::Handler`** trait.

With the lifetime-bound IRQ subsystem, the handler borrows hardware resources (like MMIO BARs) directly, eliminating `unsafe` data lookups inside the ISR.

---

## Full Example: PCI Driver with MSI Interrupts

This example builds on top of the PCI + DRM driver to add MSI interrupt registration and a simple handler (using fictitious registers for illustration on `pci-testdev`).

```rust,ignore
// SPDX-License-Identifier: GPL-2.0
//! PCI DRM driver with MSI interrupt handling.

use kernel::{
    device::{Bound, Core, DeviceContext},
    drm,
    drm::ioctl,
    drm::Registered,
    io::Io,
    irq,
    pci,
    prelude::*,
    sync::aref::ARef,
    uapi,
};

// Note: We reuse the edu DRM UAPI types here to avoid introducing new UAPI headers.
#[allow(dead_code)]
const EDU_GET_ID: u32 = kernel::ioctl::_IOR::<u32>('E' as u32, 0x00);

mod regs {
    use kernel::io::register;
    register! {
        pub(super) COUNT(u32) @ 0xC {
            31:0 count;
        }
        // Fictitious registers for illustration
        pub(super) IRQ_STATUS(u32) @ 0x10 {
            31:0 val;
        }
        pub(super) IRQ_ACKNOWLEDGE(u32) @ 0x14 {
            31:0 val;
        }
    }
    pub(super) const END: usize = 0x18;
}

struct TestPciDriver;

#[pin_data(PinnedDrop)]
struct TestPciData<'bound> {
    pdev: ARef<pci::Device>,
    _reg: drm::Registration<'bound, TestPciDriver>,
}

#[pin_data]
struct TestDrmData<'drm> {
    #[pin]
    _irq: irq::Registration<'drm, TestIrqHandler<'drm>>,
}

#[pin_data]
struct TestIrqHandler<'bound> {
    pdev: &'bound pci::Device<Bound>,
    bar: pci::Bar<'bound, { regs::END }>,
}

struct TestFile;

#[pin_data]
struct TestObject {}

impl<'bound> irq::Handler for TestIrqHandler<'bound> {
    fn handle(&self) -> irq::IrqReturn {
        let status = self.bar.read(regs::IRQ_STATUS).val().get();
        if status == 0 {
            return irq::IrqReturn::None;
        }

        dev_info!(self.pdev, "QEMU PCI testdev IRQ handled! status=0x{:x}\n", status);
        self.bar.write(regs::IRQ_ACKNOWLEDGE, status.into());

        irq::IrqReturn::Handled
    }
}

impl drm::file::DriverFile for TestFile {
    type Driver = TestPciDriver;

    fn open(_dev: &drm::Device<TestPciDriver>) -> Result<Pin<KBox<Self>>> {
        Ok(KBox::new(Self, GFP_KERNEL)?.into())
    }
}

impl TestFile {
    fn get_id(
        _dev: &drm::Device<TestPciDriver, Registered>,
        reg_data: &TestDrmData<'_>,
        arg: &mut uapi::drm_edu_get_id,
        _file: &drm::File<Self>,
    ) -> Result<u32> {
        let bar = &reg_data._irq.handler().bar;
        arg.id = bar.read(regs::COUNT).count().get();
        Ok(0)
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

#[vtable]
impl drm::Driver for TestPciDriver {
    type Data = ();
    type RegistrationData<'drm> = TestDrmData<'drm>;
    type File = TestFile;
    type Object = drm::gem::Object<TestObject>;
    type ParentDevice<Ctx: DeviceContext> = pci::Device<Ctx>;

    const INFO: drm::DriverInfo = drm::DriverInfo {
        major: 1,
        minor: 0,
        patchlevel: 0,
        name: c"pci-testdev-drm-irq",
        desc: c"PCI Testdev DRM Driver with IRQ",
    };

    const FEAT_RENDER: bool = true;

    kernel::declare_drm_ioctls! {
        (EDU_GET_ID, drm_edu_get_id, ioctl::RENDER_ALLOW, TestFile::get_id),
    }
}

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

            let bar = pdev.iomap_region_sized::<{ regs::END }>(0, c"pci_testdev_drm")?;

            let unreg_dev = drm::UnregisteredDevice::<TestPciDriver>::new(pdev, Ok(()))?;

            // Allocate 1 MSI interrupt vector
            let vectors = pdev.alloc_irq_vectors(1, 1, pci::IrqTypes::all())?;
            let vector = *vectors.start();

            // Register the interrupt handler
            let irq_init = unsafe {
                pdev.request_irq(
                    vector,
                    irq::Flags::SHARED,
                    c"pci_testdev_drm",
                    try_pin_init!(TestIrqHandler {
                        pdev: &**pdev,
                        bar,
                    }),
                )
            };

            let reg_data = try_pin_init!(TestDrmData {
                _irq <- irq_init,
            });

            let reg = unsafe {
                drm::Registration::new(pdev.as_ref(), unreg_dev, reg_data, 0)?
            };

            Ok(try_pin_init!(TestPciData {
                pdev: pdev.into(),
                _reg: reg,
            }))
        })
    }
}

#[pinned_drop]
impl PinnedDrop for TestPciData<'_> {
    fn drop(self: Pin<&mut Self>) {
        dev_info!(self.pdev, "Removing PCI Testdev DRM IRQ driver.\n");
    }
}

kernel::pci_device_table!(
    PCI_TABLE,
    <TestPciDriver as pci::Driver>::IdInfo,
    [(pci::DeviceId::from_id(pci::Vendor::REDHAT, 0x5), ())]
);

kernel::module_pci_driver! {
    type: TestPciDriver,
    name: "rust_driver_pci_testdev_drm_irq",
    authors: ["Your Name"],
    description: "PCI Testdev DRM driver with IRQ",
    license: "GPL v2",
}
```

---

## Interrupt Context Rules (The ISR)

The `handle` function runs in **interrupt context (atomic context)**:
*   **No Blocking/Sleeping:** You cannot acquire a `Mutex`, allocate memory with `GFP_KERNEL`, or perform blocking I/O inside the handler.
*   **Spinlocks:** If you need to lock shared state in an ISR, you must use `SpinLock` (never `Mutex`).
*   **Memory Safety:** When `irq::Registration` is dropped, it automatically deregisters the handler (calling `free_irq`) and blocks until any running ISR completes. This guarantees that the handler (and the borrowed `bar`) cannot be dropped while the ISR is executing.

<details>

- Discuss how `alloc_irq_vectors` configures MSI-X/MSI or legacy IRQs.
- Explain `IrqReturn::Handled` (our device triggered it) vs `IrqReturn::None` (a shared IRQ line where another device triggered it).
- Emphasize why `request_irq` is `unsafe`: leaking the returned `Registration` (e.g. via `mem::forget`) would result in the kernel keeping a dangling pointer to the handler (and the borrowed BAR) in its interrupt table after the driver is unbound, causing a crash on the next interrupt.

</details>
