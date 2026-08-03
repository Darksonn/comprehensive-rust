---
minutes: 25
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Kernel Modules

Every Rust kernel module uses the `module!` macro to declare module metadata, including its name, description, license, and entry point.

A minimal kernel module implements the `kernel::Module` trait:

```rust,ignore
use kernel::prelude::*;

module! {
    type: MyModule,
    name: "my_module",
    author: "Rust for Linux Contributors",
    description: "A minimal Rust kernel module",
    license: "GPL",
}

struct MyModule;

impl kernel::Module for MyModule {
    fn init(_module: &'static ThisModule) -> Result<Self> {
        pr_info!("MyModule initialized\n");
        Ok(MyModule)
    }
}
```

<details>

- Point out that `init` returns a `Result<Self>`, allowing initialization failures to be cleanly reported to the kernel.
- The `Module` trait implementation serves as the lifecycle entry point for the module.

</details>
