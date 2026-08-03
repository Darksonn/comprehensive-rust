---
minutes: 30
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# C FFI & Helpers

Rust for Linux integrates with existing C kernel subsystems without requiring the entire kernel to be rewritten. The FFI layer is built around two key components in the kernel source tree:

1. **`rust/bindings/bindings_helper.h`:**  
   This C header includes all kernel C headers (`<linux/fs.h>`, `<linux/slab.h>`, etc.) that need Rust FFI bindings. During the kernel build, `bindgen` parses this file to generate the `kernel::bindings::*` module.

2. **`rust/helpers/`:**  
   Because Rust FFI cannot execute C macros or call `static inline` C functions directly, the C files in `rust/helpers/` (such as `helpers.c`, `spinlock.c`, and `mutex.c`) compile exported, non-inline wrapper functions that Rust code can link against.

```rust,ignore
// Safe abstractions in the `kernel` crate wrap raw C bindings and helpers:
use kernel::bindings;

/// Safe wrapper around a kernel C function call.
pub fn get_kernel_version() -> u32 {
    // SAFETY: Calling `LINUX_VERSION_CODE` or equivalent version helper is stateless and safe.
    unsafe { bindings::LINUX_VERSION_CODE }
}
```

<details>

- Explain why `static inline` functions in C headers don't generate symbols in object files, necessitating `rust/helpers/`.
- Emphasize that module authors rarely use `kernel::bindings` directly; instead, they use safe wrappers in the `kernel` crate.

</details>
