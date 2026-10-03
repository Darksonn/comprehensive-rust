---
minutes: 10
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Failure in Rust

```rust,ignore
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

```c
struct result {
    bool success;
    union { T ok; E err; };
};
```

```rust,ignore
fn fallible() -> Result<&'static str> {
    if foo() {
        return Err(EINVAL);
    }
    Ok("my string")
}

// Early-return on error with `?`:
let string = fallible()?;
```

```text
error: unused `Result` that must be used
 --> src/main.rs:8:5
  |
8 |     fallible();
  |     ^^^^^^^^^^
  |
  = note: this `Result` may be an `Err` variant, which should be handled
```

<details>

- **Tagged union (`enum Result<T, E>`):** Explain that Rust's `enum` is a tagged union. A `Result` contains either `Ok(T)` or `Err(E)`—never both.
- **In the kernel:** `Result<T>` defaults to `Error` as the error type, which wraps negative errno values (`EINVAL`, `ENOMEM`, `ENODEV`, etc.).
- **The `?` operator:** If `fallible()` returns `Err(e)`, `?` immediately returns `Err(e)` from the enclosing function (running all local destructors on the way out). If it returns `Ok(val)`, it unwraps `val`.
- **Explicit handling:** You can also pattern-match with `match fallible() { Ok(s) => ..., Err(e) => ... }`.
- **No ignored errors:** Because `Result` is marked `#[must_use]`, forgetting `?` or `match` is a compile error.

</details>
