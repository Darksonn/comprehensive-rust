---
minutes: 10
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Building Rust Code

[`samples/rust/rust_minimal.rs`](https://github.com/Darksonn/linux/blob/rfl-course-edu/samples/rust/rust_minimal.rs):

```rust,ignore
use kernel::prelude::*;

module! {
    type: RustMinimal,
    name: "rust_minimal",
    authors: ["Rust for Linux Contributors"],
    description: "Rust minimal sample",
    license: "GPL",
}

struct RustMinimal {
    numbers: KVec<i32>,
}

impl kernel::Module for RustMinimal {
    fn init(_module: &'static ThisModule) -> Result<Self> {
        pr_info!("Rust minimal sample (init)\n");
        let mut numbers = KVec::new();
        numbers.push(72, GFP_KERNEL)?;
        numbers.push(108, GFP_KERNEL)?;
        numbers.push(200, GFP_KERNEL)?;
        Ok(RustMinimal { numbers })
    }
}

impl Drop for RustMinimal {
    fn drop(&mut self) {
        pr_info!("My numbers are {:?}\n", self.numbers);
        pr_info!("Rust minimal sample (exit)\n");
    }
}
```

[`samples/rust/Makefile`](https://github.com/Darksonn/linux/blob/rfl-course-edu/samples/rust/Makefile) & [`samples/rust/Kconfig`](https://github.com/Darksonn/linux/blob/rfl-course-edu/samples/rust/Kconfig):

```Makefile
obj-$(CONFIG_SAMPLE_RUST_MINIMAL) += rust_minimal.o
```

<details>

- **Struct vs. init function:** In C, a module specifies an init function and stores state in global variables. In Rust, a module is represented by a `struct` (`RustMinimal`) whose fields hold the module's state.
- **Traits (`impl kernel::Module`):** A trait is like a typed C vtable—a list of functions a type implements (`init` on module load, `Drop::drop` on `rmmod`).
- **Kbuild integration:** Each `.o` target in `Makefile` corresponds to a crate root compiled by `rustc`. Sub-modules within the driver are included via Rust `mod` statements.

</details>
