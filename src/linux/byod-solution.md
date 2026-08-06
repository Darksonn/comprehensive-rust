---
minutes: 10
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Solution: Build your own driver

## Part 1: Single-File Driver

`drivers/Kconfig`
```diff
--- a/drivers/Kconfig
+++ b/drivers/Kconfig
@@ -253,4 +253,6 @@ source "drivers/cdx/Kconfig"
 
 source "drivers/resctrl/Kconfig"
+
+source "drivers/alice/Kconfig"
 
 endmenu
```

`drivers/Makefile`
```diff
--- a/drivers/Makefile
+++ b/drivers/Makefile
@@ -198,3 +198,5 @@ obj-y                               += resctrl/
 
 obj-$(CONFIG_DIBS)             += dibs/
 obj-$(CONFIG_S390)             += s390/
+
+obj-y                          += alice/
```

`drivers/alice/Kconfig`
```Kconfig
# SPDX-License-Identifier: GPL-2.0

config RUST_ALICE
	tristate "Alice Rust driver"
	depends on RUST
	help
	  This option builds Alice's driver.
	
	  To compile this as a module, choose M here:
	  the module will be called rust_alice.
	
	  If unsure, say N.
```

`drivers/alice/Makefile`
```Makefile
# SPDX-License-Identifier: GPL-2.0

obj-$(CONFIG_RUST_ALICE) += rust_alice.o
```

`drivers/alice/rust_alice.rs`
```rust,ignore
// SPDX-License-Identifier: GPL-2.0

//! Alice's Rust driver.

use kernel::prelude::*;

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

        let mut numbers = KVec::new();
        numbers.push(72, GFP_KERNEL)?;
        numbers.push(108, GFP_KERNEL)?;
        numbers.push(200, GFP_KERNEL)?;

        Ok(RustAlice { numbers })
    }
}

impl Drop for RustAlice {
    fn drop(&mut self) {
        pr_info!("Alice's numbers are {:?}\n", self.numbers);
    }
}
```

## Part 2 (Stretch Goal): Multi-File Driver

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

<details>

- Discuss why `depends on RUST` is needed in Kconfig.
- Explain how Rust's module system (`mod numbers;`) integrates with kbuild: you do **not** add `numbers.o` to `obj-m` in the Makefile, because `rustc` compiles the entire crate starting from the root file (`rust_alice.rs`).

</details>
