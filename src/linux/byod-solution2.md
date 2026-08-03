---
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Adding a second file

`drivers/alice/Makefile`
```Makefile
# SPDX-License-Identifier: GPL-2.0

obj-$(CONFIG_RUST_ALICE) += rust_alice.o
```
`drivers/alice/rust_alice.rs`
```rust,ignore
// SPDX-License-Identifier: GPL-2.0

//! Alice's Rust driver.

use crate::numbers::my_numbers;
use kernel::prelude::*;

mod numbers;

module! {
    type: RustAlice,
    name: "rust_alice",
    authors: ["Alice Ryhl"],
    description: "Alice's driver",
    license: "GPL",
}

struct RustAlice {
    numbers: KVec<i32>,
}

impl kernel::Module for RustAlice {
    fn init(_module: &'static ThisModule) -> Result<Self> {
        pr_info!("Alice's driver is being loaded\n");
        pr_info!("Am I built-in? {}\n", !cfg!(MODULE));

        Ok(RustAlice {
            numbers: my_numbers()?,
        })
    }
}

impl Drop for RustAlice {
    fn drop(&mut self) {
        pr_info!("Alice's numbers are {:?}\n", self.numbers);
    }
}
```
`drivers/alice/numbers.rs`
```rust,ignore
// SPDX-License-Identifier: GPL-2.0

use kernel::prelude::*;

pub(crate) fn my_numbers() -> Result<KVec<i32>> {
    let mut numbers = KVec::new();
    numbers.push(72, GFP_KERNEL)?;
    numbers.push(108, GFP_KERNEL)?;
    numbers.push(200, GFP_KERNEL)?;
    Ok(numbers)
}
```

