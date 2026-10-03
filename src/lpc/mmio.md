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
    use kernel::io::register;
    register! {
        pub(super) DATA(u8) @ 0x08 {
            7:0 data;
        }
        pub(super) COUNT(u32) @ 0x0C {
            31:0 count;
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
        reg_data.bar.write(regs::DATA, 5.into());
        arg.id = reg_data.bar.read(regs::COUNT).count().get();
        Ok(0)
    }
}
```

<details>

- **`iomap_region_sized::<{ regs::END }>`:** In `probe()` (already set up in our starter stub), the driver checks at runtime that PCI BAR 0 is at least `regs::END` (`0x80`) bytes long, and maps it into kernel virtual memory as `pci::Bar<'drm, { regs::END }>`.
- **Compile-time bounds checking (`register!`):** Every register defined in `register!` has its offset checked at compile time against `{ regs::END }`, eliminating out-of-bounds MMIO reads/writes and manual bitmask shift errors.
- **Ways to read and write registers:**
  - **Reading a field:** `bar.read(regs::COUNT)` returns a `regs::COUNT` value; `.count()` extracts the field as a `Bounded<u32, 32>`, and `.get()` (or `*` / `.into()`) returns the inner `u32`. You can also call `.into_raw()` on the register itself to get the raw `u32` directly.
  - **Writing a whole register vs. bitfields:** Because `regs::DATA` implements `From<u8>`, `5.into()` (or `regs::DATA::from_raw(5)`) constructs the whole register from a raw integer. For multi-field registers, you can use the builder pattern `regs::DATA::zeroed().with_data(5)` and write it with `bar.write(regs::DATA, ...)` or `bar.write_reg(...)`.
  - **Raw offsets (without `register!`):** `Io` also provides `bar.read32(offset)` / `bar.write32(val, offset)` (compile-time bounds checked) and `bar.try_read32(offset)?` / `bar.try_write32(val, offset)?` (runtime bounds checked).

</details>
