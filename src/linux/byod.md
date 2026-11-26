---
minutes: 25
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Exercise: Build Your Own Driver

First, update your `~/learn-rust/linux` checkout to the latest `rfl-course-edu`
branch (see [Setup](setup.md)):

<!-- mdbook-xgettext: skip -->

```bash
cd ~/learn-rust/linux && git fetch origin && git reset --hard origin/rfl-course-edu
```

**Part 1:** Define your own module and build it.

1. Create a folder called `drivers/<your name>/`. In this example we will use
   `drivers/alice/`.
2. Add a Kconfig entry for your driver:
   - Create `drivers/alice/Kconfig` with a config option (`config RUST_ALICE`)
     for your driver.
   - Add `source "drivers/alice/Kconfig"` to `drivers/Kconfig`.
3. Wire your driver into kbuild:
   - Create `drivers/alice/Makefile` with a line to build your new driver.
   - Add `obj-y += alice/` to `drivers/Makefile`.
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
use kernel::prelude::*;

pub(crate) fn my_numbers() -> Result<KVec<i32>> {
    let mut numbers = KVec::new();
    numbers.push(72, GFP_KERNEL)?;
    numbers.push(108, GFP_KERNEL)?;
    numbers.push(200, GFP_KERNEL)?;
    Ok(numbers)
}
```

<details>

- **Common pitfalls:**
  - Forgetting `make LLVM=1 olddefconfig` after `./scripts/config --module CONFIG_RUST_ALICE` (the new symbol is not written cleanly into `.config` until `olddefconfig` runs).
  - Adding `numbers.o` to `drivers/alice/Makefile` in Part 2—`rustc` compiles the whole crate from `rust_alice.rs` via `mod numbers;`, so only `rust_alice.o` belongs in the Makefile.
  - Vermagic mismatch on `insmod` if `CONFIG_LOCALVERSION_AUTO` was left enabled; with it disabled (as in Setup), rebuilding a module does not require rebooting QEMU.
- **When reviewing the solution:** Point out `depends on RUST` in `drivers/alice/Kconfig`—it won't block building here since `CONFIG_RUST=y` is already enabled, and `SAMPLE_RUST_MINIMAL` didn't need it directly because `menuconfig SAMPLES_RUST` wraps it in `depends on RUST`, but real drivers need it.

</details>
