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

```rust,ignore
use kernel::pci;
use kernel::io::{register, Io};

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

fn init_mmio<'bound>(
    pdev: &'bound pci::Device<Core<'_>>,
) -> Result<pci::Bar<'bound, { regs::END }>> {
    // 1. Enable memory space access (maps the physical BAR decoder):
    pdev.enable_device_mem()?;

    // 2. Map BAR 0 (the MMIO range of size 0x10):
    // Returns a Bar<'bound, SIZE> which borrows the device.
    let bar = pdev.iomap_region_sized::<{ regs::END }>(0, c"pci_testdev")?;

    // 3. Read and write registers directly (no runtime lock needed):
    let count = bar.read(regs::COUNT).count().get();
    dev_info!(pdev, "Test count: {}\n", count);

    bar.write(regs::DATA, 5.into());
    
    Ok(bar)
}
```

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
