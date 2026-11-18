---
minutes: 10
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Memory-Mapped I/O

```rust,ignore
mod regs {
    use kernel::io::{register, Region};

    register! {
        base: Region<END>;

        pub(super) DATA: u8 @ 0x08;
        pub(super) COUNT: u32 @ 0x0C;
        pub(super) STATUS(u32) @ 0x20 {
            7:4 mode;
            0:0 busy;
        }
    }
    pub(super) const END: usize = 0x80;
}

#[pin_data]
struct EduDrmData<'drm> {
    bar: pci::Bar<'drm, { regs::END }>,
}
```

Reading and writing registers via `reg_data.bar`:

```rust,ignore
impl EduFile {
    fn get_id(
        _dev: &drm::Device<EduDriver, Registered>,
        reg_data: &EduDrmData<'_>,
        arg: &mut uapi::drm_edu_get_id,
        _file: &drm::File<Self>,
    ) -> Result<u32> {
        reg_data.bar.write(regs::DATA, 5);
        arg.id = reg_data.bar.read(regs::COUNT);
        Ok(0)
    }
}
```

<details>

- **`iomap_region_sized::<{ regs::END }>`:** In `probe()` (already set up in our starter stub), the driver checks at runtime that PCI BAR 0 is at least `regs::END` (`0x80`) bytes long, and maps it into kernel virtual memory as `pci::Bar<'drm, { regs::END }>`.
- **Compile-time bounds checking (`register!`):** Every register defined in `register!` has its offset checked at compile time against `Region<END>`, eliminating out-of-bounds MMIO reads/writes and manual bitmask shift errors.
- **Plain typed registers vs. bitfield registers:**
  - **Plain typed registers (`NAME: Type @ offset;`):** Used when a register holds a single integer value (such as `DATA: u8 @ 0x08;` or `COUNT: u32 @ 0x0C;`). `bar.read(regs::COUNT)` returns `u32` directly, and `bar.write(regs::DATA, 5)` takes `u8` directly.
  - **Bitfield registers (`NAME(Type) @ offset { ... }`):** Used when a register packs multiple bitfields (such as `STATUS(u32) @ 0x20`). `bar.read(regs::STATUS)` returns a `regs::STATUS` bitfield struct whose fields are accessed via `.busy().get()` or `.mode().get()`.
  - **Raw offsets (without `register!`):** `Io` also provides `bar.read32(offset)` / `bar.write32(val, offset)` (compile-time bounds checked) and `bar.try_read32(offset)?` / `bar.try_write32(val, offset)?` (runtime bounds checked).

</details>
