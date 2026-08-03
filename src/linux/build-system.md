---
minutes: 35
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# The kernel build system and Rust

In the setup section, you built the minimal Rust sample. Let's understand how
the build system actually works and how this sample is built.

The sample can be found at `samples/rust/rust_minimal.rs`.
```rust,ignore
// SPDX-License-Identifier: GPL-2.0

//! Rust minimal sample.

use kernel::prelude::*;

module! {
    type: RustMinimal,
    name: "rust_minimal",
    authors: ["Rust for Linux Contributors"],
    description: "Rust minimal sample",
    license: "GPL",
    params: {
        test_parameter: i64 {
            default: 1,
            description: "This parameter has a default of 1",
        },
    },
}

struct RustMinimal {
    numbers: KVec<i32>,
}

impl kernel::Module for RustMinimal {
    fn init(_module: &'static ThisModule) -> Result<Self> {
        pr_info!("Rust minimal sample (init)\n");
        pr_info!("Am I built-in? {}\n", !cfg!(MODULE));
        pr_info!(
            "test_parameter: {}\n",
            *module_parameters::test_parameter.value()
        );

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

This file is included in the build because `samples/rust/Makefile` contains:
```Makefile
obj-$(CONFIG_SAMPLE_RUST_MINIMAL) += rust_minimal.o
```

And in `samples/rust/Kconfig` you will find:
```kconfig
config SAMPLE_RUST_MINIMAL
	tristate "Minimal"
	help
	  This option builds the Rust minimal module sample.

	  To compile this as a module, choose M here:
	  the module will be called rust_minimal.

	  If unsure, say N.
```

<details>

- Things to discuss:
  - Only one `module!` macro per driver.
  - Meaning of `tristate` and loading/unloading drivers.
  - Kconfig and Makefile are separate systems.
  - The Makefile corresponds to invocations to `rustc`. Sub-modules are included via `mod` statements.

</details>
