---
minutes: 25
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Exercise: Build Your Own Driver

First, make sure your `~/learn-rust/linux` checkout is up to date on the `rfl-course-edu` branch and that DRM and the afternoon sample driver are enabled in your `.config` (see [Setup](setup.md) for full details):

```bash
cd ~/learn-rust/linux
git pull
./scripts/config --enable CONFIG_DRM
./scripts/config --module SAMPLE_RUST_DRIVER_PCI_EDU_DRM
```

**Part 1:** Define your own module and build it.

1. Create a folder called `drivers/<your name>/`. In this example we will use
   `drivers/alice/`.
2. Create `drivers/alice/Kconfig` with a config option (`CONFIG_RUST_ALICE`) for
   your driver, and add `source "drivers/alice/Kconfig"` to `drivers/Kconfig`.
3. Create `drivers/alice/Makefile` with a line to build your new driver, and
   add `obj-y += alice/` to `drivers/Makefile`.
4. Define a `drivers/alice/rust_alice.rs` file containing a Rust module (using
   [`samples/rust/rust_minimal.rs`](https://github.com/Darksonn/linux/blob/rfl-course-edu/samples/rust/rust_minimal.rs)
   as a starting point).
5. Enable your driver as a module and build the kernel:
   ```bash
   ./scripts/config --module CONFIG_RUST_ALICE
   make LLVM=1 olddefconfig
   make LLVM=1 -j$(nproc)
   ```
6. Boot QEMU (see [Setup](setup.md)) and load/unload your driver with
   `insmod /mnt/linux/drivers/alice/rust_alice.ko` and `rmmod rust_alice`.

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
