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
    pub(super) STATUS(u32) @ 0x20 {
        7:4 mode;
        0:0 busy;
    }
}
```

Expands to both a **location constant** and a **bitfield struct** in `mod regs`:

```rust,ignore
pub(super) const STATUS: FixedRegisterLoc<STATUS> = FixedRegisterLoc::new();

#[repr(transparent)]
#[derive(Clone, Copy, PartialEq, Eq)]
pub(super) struct STATUS(u32);

impl Register for STATUS {
    type Storage = u32;
    const OFFSET: usize = 0x20;
}
impl FixedRegister for STATUS {}
impl From<u32> for STATUS { ... }
impl From<STATUS> for u32 { ... }

impl STATUS {
    pub(super) const fn from_raw(value: u32) -> Self { Self(value) }
    pub(super) const fn into_raw(self) -> u32 { self.0 }

    pub(super) fn mode(self) -> Bounded<u32, 4> { ... }
    pub(super) fn with_mode<T: Into<Bounded<u32, 4>>>(self, val: T) -> Self { ... }
    pub(super) const fn with_const_mode<const V: u32>(self) -> Self { ... }
    pub(super) fn try_with_mode<T: TryIntoBounded<u32, 4>>(self, val: T) -> Result<Self> { ... }

    pub(super) fn busy(self) -> Bounded<u32, 1> { ... }
    pub(super) fn with_busy<T: Into<Bounded<u32, 1>>>(self, val: T) -> Self { ... }
}
```

<details>

- **Dual namespace trick (`const STATUS` + `struct STATUS`):**
  - When you write `bar.read(regs::STATUS)`, `regs::STATUS` refers to the `const STATUS: FixedRegisterLoc<STATUS>`, which tells `Io::read` both the offset (`0x20`) and the return type (`struct STATUS`).
- **`Bounded<T, N>`:**
  - Field getters return `Bounded<u32, N>`, which statically guarantees the value uses at most `N` bits.
  - Call `.get()` (or dereference `*`, or `.into()`) to extract the `u32`, or compare directly (`status.busy() == 0`). For 1-bit fields, `.into_bool()` converts to `bool`.
- **Constructing values to write:**
  - **Whole register:** `bar.write(regs::STATUS, raw_u32.into())` uses `From<u32> for STATUS`.
  - **Bitfield builders:** `STATUS::zeroed().with_busy(true).with_const_mode::<3>()` sets individual bit ranges without manual shifts or masks. Because `STATUS` implements `FixedRegister`, you can write it with either `bar.write(regs::STATUS, val)` or the single-argument `bar.write_reg(val)`.

</details>
