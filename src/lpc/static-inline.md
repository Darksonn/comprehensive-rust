---
minutes: 10
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Static Inline Functions

[`include/linux/cred.h`](https://github.com/Darksonn/linux/blob/rfl-course-edu/include/linux/cred.h):

```c
static inline const struct cred *get_cred(const struct cred *cred)
{
    if (cred)
        atomic_long_inc(&((struct cred *)cred)->usage);
    return cred;
}
```

[`rust/helpers/cred.c`](https://github.com/Darksonn/linux/blob/rfl-course-edu/rust/helpers/cred.c):

```c
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

<details>

- **The linker problem:** Many C kernel APIs (like `get_cred()` and `put_cred()`) are defined in headers as `static inline` functions or C macros. Because `static inline` functions are inlined into C callers and do not emit a linker symbol, `bindgen` cannot link against `get_cred` directly.
- **The solution (`rust/helpers/*.c`):** We write small C wrapper functions marked `__rust_helper` in `rust/helpers/`.
  1. Kbuild compiles `rust/helpers/` into object files exporting symbols (or inlines them across languages when LTO is enabled).
  2. `bindgen` strips the `rust_helper_` prefix so Rust code can call `bindings::get_cred(...)` and `bindings::put_cred(...)`.

</details>
