---
minutes: 30
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Kernel Abstractions

Rust for Linux provides safe abstractions wrapping underlying C kernel APIs in the `kernel` crate.

- **Error Handling:** Kernel error codes (`-ENOMEM`, `-EINVAL`) are represented by the `kernel::error::Error` type and integrated with Rust's standard `Result<T>` and `?` operator.
- **Memory Allocations:** Kernel data structures use specialized allocators aware of GFP (Get Free Page) flags.

```rust,ignore
use kernel::prelude::*;

fn do_something() -> Result<()> {
    // Operations returning Result automatically map to kernel error codes on failure.
    Ok(())
}
```

<details>

- Explain that C functions returning integer error codes are wrapped in safe Rust functions returning `Result<T>`.
- Discuss how the `?` operator simplifies propagating error codes back to the kernel.

</details>
