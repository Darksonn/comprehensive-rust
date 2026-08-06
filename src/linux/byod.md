---
minutes: 30
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Exercise: Build your own driver

**Exercise:** Define your own module and build it.

1. Create a folder called `drivers/<your name>/`. In this example we will use
   `drivers/alice/`.
2. Create `drivers/alice/Kconfig` with a config option for your driver, and add
   it to `drivers/Kconfig`.
3. Create `drivers/alice/Makefile` with a line to build your new driver, and
   add it to `drivers/Makefile`.
4. Define a `drivers/alice/rust_alice.rs` file containing a Rust module.
5. Build the kernel with your driver.
6. Boot QEMU and load your driver.
