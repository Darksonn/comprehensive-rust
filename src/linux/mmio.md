---
minutes: 15
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Memory-Mapped I/O (MMIO)

PCI devices expose their hardware registers via Base Address Registers (BARs). We map these BARs into kernel virtual memory to read and write registers.

In Rust, a mapped BAR is represented by the **`pci::Bar`** struct. It is parameterized by a lifetime that binds it to the device's probe scope, ensuring the CPU cannot access registers after the device is unbound.

---

## Full Example: DRM PCI Driver with MMIO

This example integrates MMIO into our DRM driver. We map **BAR 0** during `probe`, share it with the DRM subsystem using `TestDrmData`, and update the `GET_ID` IOCTL to read the `COUNT` register from the hardware.

```rust,ignore
// SPDX-License-Identifier: GPL-2.0
//! DRM PCI driver with MMIO register access.

use kernel::{
    device::{Core, DeviceContext},
    drm,
    drm::ioctl,
    drm::Registered,
    io::Io,
    pci,
    prelude::*,
    sync::aref::ARef,
    uapi,
};

mod regs {
    use kernel::io::register;
    register! {
        pub(super) DATA(u8) @ 0x8 {
            7:0 data;
        }
        pub(super) COUNT(u32) @ 0xC {
            31:0 count;
        }
    }
    pub(super) const END: usize = 0x10;
}

struct TestPciDriver;

#[pin_data]
struct TestPciData<'bound> {
    pdev: ARef<pci::Device>,
    _reg: drm::Registration<'bound, TestPciDriver>,
}

// 1. Store the mapped BAR in the shared registration data:
#[pin_data]
struct TestDrmData<'drm> {
    bar: pci::Bar<'drm, { regs::END }>,
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

            // 2. Map BAR 0 (the MMIO range of size 0x10):
            let bar = pdev.iomap_region_sized::<{ regs::END }>(0, c"pci_testdev_drm")?;

            // We can write to registers in probe for testing:
            bar.write(regs::DATA, 5.into());

            let unreg_dev = drm::UnregisteredDevice::<TestPciDriver>::new(pdev, Ok(()))?;

            // Pass the Bar to the shared DRM registration data:
            let reg_data = try_pin_init!(TestDrmData {
                bar,
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
        name: c"pci-testdev-drm",
        desc: c"PCI Testdev DRM Driver",
    };

    const FEAT_RENDER: bool = true;

    kernel::declare_drm_ioctls! {
        (EDU_GET_ID, drm_edu_get_id, ioctl::RENDER_ALLOW, TestFile::get_id),
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
        // 3. Access the BAR from the shared registration data:
        arg.id = reg_data.bar.read(regs::COUNT).count().get();
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

kernel::pci_device_table!(
    PCI_TABLE,
    <TestPciDriver as pci::Driver>::IdInfo,
    [(pci::DeviceId::from_id(pci::Vendor::REDHAT, 0x5), ())]
);

kernel::module_pci_driver! {
    type: TestPciDriver,
    name: "rust_driver_pci_testdev_drm",
    authors: ["Your Name"],
    description: "PCI Testdev DRM driver with MMIO",
    license: "GPL v2",
}
```

---

## Verifying the Integration

If you compile and load this driver, you can run the `test_ioctl` program compiled in the [Defining DRM IOCTLs](drm-ioctl.md) slide:

```bash
# Run the test binary again:
./test_ioctl
```

Instead of the dummy `0x12345678` value, it should now print the actual hardware count value read from the register.

---

## Safe MMIO Invariants

- **`iomap_region_sized`:** Safe mapping ensures the memory range does not exceed the BAR size, preventing out-of-bounds reads/writes.
- **Lifetime Binding (`'bound`):** The `Bar` borrows the `pci::Device`. The compiler ensures that this `Bar` cannot be stored or used past the unbind phase of the device.
- **The `register!` Macro:** Provides type-safe register definitions with field-level bitmasks, preventing invalid bit writes.
- **Memory Barriers:** Register accessors invoke hardware barriers (such as `mb()`, `rmb()`, `wmb()`) under the hood to ensure writes to the device are not reordered by the CPU pipeline or compiler.

<details>

- Explain why `pci::Bar` can be used directly: since we can use lifetime-bound driver data, we can store `Bar<'bound>` directly.
- Mention `into_devres()` as an alternative: if a driver *must* use a `'static` driver data structure (like the miscdevice-based PCI driver), it can convert the `Bar` into a `DevresBar` (which is `DevresLt<Bar<'static>>`). This requires runtime borrow checking using `.try_access()`.
- Compare C-style pointer dereferencing (`writel(val, addr)`) with Rust's structured `bar.write` API.

</details>
