---
minutes: 15
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Memory-Mapped I/O (MMIO)

PCI devices expose their hardware registers via Base Address Registers (BARs). We map these BARs into kernel virtual memory to read and write registers.

```rust,ignore
use kernel::pci;
use kernel::io::{register, Io};

mod regs {
    register! {
        pub(super) ID(u32) @ 0x00 {
            31:0 id;
        }
        pub(super) FACTORIAL(u32) @ 0x08 {
            31:0 val;
        }
        pub(super) STATUS(u32) @ 0x20 {
            0:0 computing;
        }
    }
    pub(super) const END: usize = 0x80;
}

fn init_mmio(pdev: &pci::Device<Core<'_>>) -> Result {
    // 1. Enable memory space access (maps the physical BAR decoder):
    pdev.enable_device_mem()?;

    // 2. Map BAR 0 (the MMIO range of size 0x80):
    let bar: pci::DevresBar<{ regs::END }> = pdev
        .iomap_region_sized::<{ regs::END }>(0, c"qemu_edu")?
        .into_devres()?;

    // 3. Read and write registers:
    let bar_access = bar.try_access().ok_or(ENODEV)?;
    
    let id = bar_access.read(regs::ID).id();
    dev_info!(pdev, "Device ID: 0x{:x}\n", id);

    bar_access.write_reg(regs::FACTORIAL::zeroed().with_val(5));
    
    Ok(())
}
```

## Safe MMIO Invariants

- **`iomap_region_sized`:** Safe mapping ensures the memory range does not exceed the BAR size, preventing out-of-bounds reads/writes.
- **The `register!` Macro:** Provides type-safe register definitions with field-level bitmasks, preventing invalid bit writes.
- **Memory Barriers:** Register accessors invoke hardware barriers (such as `mb()`, `rmb()`, `wmb()`) under the hood to ensure writes to the device are not reordered by the CPU pipeline or compiler.

<details>

- Explain why `DevresBar` is used: it ties the life of the mapped virtual memory region to the driver's device presence (devres resource manager).
- Compare C-style pointer dereferencing (`writel(val, addr)`) with Rust's structured `bar.write_reg` API.

</details>
