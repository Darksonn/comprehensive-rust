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

This example builds on top of the minimal DRM driver. We add a register block, map **BAR 0** during `probe`, share it with the DRM registration, and implement a `GET_ID` IOCTL that reads the `COUNT` register.

```rust,ignore
// SPDX-License-Identifier: GPL-2.0
//! DRM PCI driver with MMIO register access.

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

// Note: We reuse the edu DRM UAPI types here to avoid introducing new UAPI headers.
#[allow(dead_code)]
const EDU_GET_ID: u32 = kernel::ioctl::_IOR::<u32>('E' as u32, 0x00);

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
            // 1. Enable memory space access (maps the physical BAR decoder):
            pdev.enable_device_mem()?;
            pdev.set_master();

            // 2. Map BAR 0 (the MMIO range of size 0x10):
            // Returns a Bar<'bound, SIZE> which borrows the device.
            let bar = pdev.iomap_region_sized::<{ regs::END }>(0, c"pci_testdev_drm")?;

            // 3. Read and write registers directly (no runtime lock needed):
            let count = bar.read(regs::COUNT).count().get();
            dev_info!(pdev, "Initial test count: {}\n", count);

            bar.write(regs::DATA, 5.into());

            let unreg_dev = drm::UnregisteredDevice::<TestPciDriver>::new(pdev, Ok(()))?;

            // 4. Store the Bar inside TestDrmData to share it with DRM file operations.
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
        _dev: &drm::Device<TestPciDriver, drm::Registered>,
        reg_data: &TestDrmData<'_>,
        arg: &mut uapi::drm_edu_get_id,
        _file: &drm::File<Self>,
    ) -> Result<u32> {
        // Access the BAR from the shared registration data:
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

## Safe MMIO Invariants

- **`iomap_region_sized`:** Safe mapping ensures the memory range does not exceed the BAR size, preventing out-of-bounds reads/writes.
- **Lifetime Binding (`'bound`):** The `Bar` borrows the `pci::Device`. The compiler ensures that this `Bar` cannot be stored or used past the unbind phase of the device.
- **The `register!` Macro:** Provides type-safe register definitions with field-level bitmasks, preventing invalid bit writes.
- **Memory Barriers:** Register accessors invoke hardware barriers (such as `mb()`, `rmb()`, `wmb()`) under the hood to ensure writes to the device are not reordered by the CPU pipeline or compiler.

---

## Userspace Interaction: Calling the IOCTL

To verify our DRM driver works and we can read the register from userspace, we can write a simple C program that opens the DRM render node and calls our `EDU_GET_ID` IOCTL.

Save the following code as `test_ioctl.c`:

```c
#include <fcntl.h>
#include <stdio.h>
#include <sys/ioctl.h>
#include <unistd.h>
#include <drm/drm.h>

struct drm_edu_get_id {
    __u32 id;
};

#define DRM_EDU_GET_ID             0x00
#define DRM_IOCTL_EDU_GET_ID       DRM_IOR(DRM_COMMAND_BASE + DRM_EDU_GET_ID, struct drm_edu_get_id)

int main() {
    // Open the render node (does not require root/master privileges)
    int fd = open("/dev/dri/renderD128", O_RDWR);
    if (fd < 0) {
        perror("Failed to open /dev/dri/renderD128");
        return 1;
    }

    struct drm_edu_get_id arg = {0};
    if (ioctl(fd, DRM_IOCTL_EDU_GET_ID, &arg) < 0) {
        perror("IOCTL failed");
        close(fd);
        return 1;
    }

    printf("Device register value (COUNT): %u\n", arg.id);
    close(fd);
    return 0;
}
```

### Compiling and Running

You can compile this program on your host machine statically, copy it to the virtual machine, and run it.

```bash
# Compile statically on host:
gcc -static -o test_ioctl test_ioctl.c

# Run inside the VM (assuming your driver is loaded):
./test_ioctl
```

<details>

- Explain why `pci::Bar` can be used directly: since we can use lifetime-bound driver data, we can store `Bar<'bound>` directly.
- Mention `into_devres()` as an alternative: if a driver *must* use a `'static` driver data structure (like the miscdevice-based PCI driver), it can convert the `Bar` into a `DevresBar` (which is `DevresLt<Bar<'static>>`). This requires runtime borrow checking using `.try_access()`.
- Compare C-style pointer dereferencing (`writel(val, addr)`) with Rust's structured `bar.write` API.

</details>
