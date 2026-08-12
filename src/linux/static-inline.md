---
minutes: 10
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Static Inline Functions

Many critical C kernel APIs are defined directly in header files as **`static inline`** functions (for example, `get_cred()` and `put_cred()` in `include/linux/cred.h`):

```c
// include/linux/cred.h
static inline const struct cred *get_cred(const struct cred *cred) {
    if (cred) {
        // Increment reference count...
    }
    return cred;
}
```

## The Linker Problem

`static inline` functions are compiled directly into individual C object files and **do not export a linker symbol**.

Because there is no symbol in the compiled kernel for `get_cred`, Rust code cannot call `bindings::get_cred()` directly at link time.

## The Solution: C Helpers (`rust/helpers/cred.c`)

Rust for Linux solves this by providing non-inlined C wrapper functions in the **`rust/helpers/`** directory.

For credentials, **`rust/helpers/cred.c`** defines:

```c
// rust/helpers/cred.c
#include <linux/cred.h>

__rust_helper const struct cred *rust_helper_get_cred(const struct cred *cred)
{
	return get_cred(cred);
}

__rust_helper void rust_helper_put_cred(const struct cred *cred)
{
	put_cred(cred);
}
```

1. Kbuild compiles `rust/helpers/cred.c` into a regular C object file, exporting the symbol `rust_helper_get_cred`.
2. `bindgen` generates FFI declarations for `rust_helper_get_cred` in `kernel::bindings`.
3. The `kernel` crate wraps the helper inside the safe `AlwaysRefCounted` implementation.

<details>

- Whenever you need to call a C `static inline` function from Rust and it is missing from `kernel::bindings`, add a wrapper to `rust/helpers/`.
- During link-time optimization (LTO), the compiler can still inline the helper across the language boundary.

</details>
