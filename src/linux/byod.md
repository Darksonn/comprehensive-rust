---
minutes: 30
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Exercise: Build your own driver

**Part 1:** Define your own module and build it.

1. Create a folder called `drivers/<your name>/`. In this example we will use
   `drivers/alice/`.
2. Create `drivers/alice/Kconfig` with a config option for your driver, and add
   it to `drivers/Kconfig`.
3. Create `drivers/alice/Makefile` with a line to build your new driver, and
   add it to `drivers/Makefile`.
4. Define a `drivers/alice/rust_alice.rs` file containing a Rust module.
5. Build the kernel with your driver.
6. Boot QEMU and load your driver with `insmod`.

**Part 2 (Stretch Goal): Add a second file**

Create a second file `drivers/alice/numbers.rs` containing a function that
creates the `KVec<i32>` value. Include it in your module using `mod numbers;` in
`rust_alice.rs` to build a multi-file driver:

```rust,ignore
pub(crate) fn my_numbers() -> Result<KVec<i32>> {
    let mut numbers = KVec::new();
    numbers.push(72, GFP_KERNEL)?;
    numbers.push(108, GFP_KERNEL)?;
    numbers.push(200, GFP_KERNEL)?;
    Ok(numbers)
}
```
