---
minutes: 25
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Kernel Error Handling

In Linux C code, functions typically signal failure by returning negative integer error codes (e.g., `-ENOMEM`, `-EINVAL`, `-EIO`).

Rust for Linux integrates kernel error codes with Rust's standard error handling:

- **`kernel::error::Error`:** Represents a negative Linux errno code.
- **`Result<T>`:** Type alias for `core::result::Result<T, kernel::error::Error>`.
- **`?` Operator:** Automatically propagates and converts kernel errors.
- **`#[must_use]`:** Unlike C, the Rust compiler emits a warning if a caller ignores a `Result`.

```rust,ignore
use kernel::prelude::*;

/// Example of fallible kernel logic using the `?` operator:
fn allocate_and_configure() -> Result<u32> {
    // If an allocation or helper call fails, `?` returns early with the kernel errno:
    let value = parse_config()?;
    if value == 0 {
        return Err(EINVAL);
    }
    Ok(value)
}

fn parse_config() -> Result<u32> {
    Ok(42)
}
```

<details>

- **Instructor Demo:** In class, delete the `?` or ignore a `Result` call in a sample module and run `make LLVM=1 samples/rust/` to show the compiler's `unused Result that must be used` warning.
- Discuss how this prevents a common class of C kernel bugs where unchecked return codes lead to undefined behavior or silent failures.

</details>
