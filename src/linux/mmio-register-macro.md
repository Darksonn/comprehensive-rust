---
minutes: 5
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Expanding the `register!` Macro

```rust,ignore
register! {
    base: Region<END>;

    pub(super) COUNT: u32 @ 0x0C;
    pub(super) STATUS(u32) @ 0x20 {
        7:4 mode;
        0:0 busy;
    }
}
```

Expands to **location constants** (and a **bitfield struct** when fields are specified):

```rust,ignore
// Plain typed register: just an OffsetLoc constant for `u32`.
pub(super) const COUNT: OffsetLoc<Region<END>, u32> = OffsetLoc::new(0x0C);

// Bitfield register: both an OffsetLoc constant and a bitfield struct.
pub(super) const STATUS: OffsetLoc<Region<END>, STATUS> = OffsetLoc::new(0x20);

#[repr(transparent)]
#[derive(Clone, Copy, PartialEq, Eq)]
pub(super) struct STATUS(u32);

impl FixedIoLoc<Region<END>> for STATUS {
    type Location = OffsetLoc<Region<END>, STATUS>;
    const LOCATION: Self::Location = STATUS;
}

impl STATUS {
    pub(super) const fn from_raw(value: u32) -> Self { Self(value) }
    pub(super) const fn into_raw(self) -> u32 { self.0 }

    pub(super) fn mode(self) -> Bounded<u32, 4> { ... }
    pub(super) fn with_mode<T: Into<Bounded<u32, 4>>>(self, val: T) -> Self { ... }
    pub(super) const fn with_const_mode<const V: u32>(self) -> Self { ... }

    pub(super) fn busy(self) -> Bounded<u32, 1> { ... }
    pub(super) fn with_busy<T: Into<Bounded<u32, 1>>>(self, val: T) -> Self { ... }
}
```

<details>

- **Plain typed registers (`COUNT: u32 @ 0x0C;`):**
  - Generates only `const COUNT: OffsetLoc<Region<END>, u32> = OffsetLoc::new(0x0C);`.
  - `bar.read(regs::COUNT)` returns `u32` directly and `bar.write(regs::COUNT, val)` takes `u32` directly.
- **Bitfield registers (`STATUS(u32) @ 0x20 { ... }`):**
  - Uses Rust's separate value and type namespaces: `const STATUS` is the `OffsetLoc<Region<END>, STATUS>` passed to `bar.read(regs::STATUS)`, while `struct STATUS(u32)` is the returned bitfield value.
- **`Bounded<T, N>`:**
  - Bitfield getters return `Bounded<u32, N>`, which statically guarantees the value uses at most `N` bits.
  - Call `.get()` (or dereference `*`, or `.into()`) to extract the `u32`, or compare directly (`status.busy() == 0`). For 1-bit fields, `.into_bool()` converts to `bool`.
- **Constructing bitfield values to write:**
  - `STATUS::zeroed().with_busy(true).with_const_mode::<3>()` sets individual bit ranges without manual shifts or masks. Because `STATUS` implements `FixedIoLoc`, you can write it with either `bar.write(regs::STATUS, val)` or the single-argument `bar.write_reg(val)`.

</details>
