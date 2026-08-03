---
minutes: 40
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Pinned Initialization

Many Linux kernel data structures contain embedded spinlocks, mutexes, or linked-list headers that are self-referential. In C, initializing these structures requires passing their memory address to an init function (e.g., `mutex_init(&lock)`), meaning the structure **must never move in memory** after initialization.

To solve this without overflowing the small kernel stack (typically 8 KB–16 KB), Rust for Linux uses the `pin-init` crate:

- **`PinInit<T, E>`:** Represents an initializer that writes a value directly into a pinned memory slot without creating an intermediate stack copy.
- **`pin_init!` and `try_pin_init!`:** Macros for defining in-place struct initializers.
- **In-Place Heap Allocation:** `KBox::pin_init` and `Arc::pin_init` allocate raw heap memory first, then run the initializer in place.

```rust,ignore
use kernel::prelude::*;
use kernel::sync::Mutex;

struct MyData {
    value: usize,
    lock: Mutex<i32>,
}

impl MyData {
    // Returns a pinned initializer rather than a stack value:
    fn new() -> impl PinInit<Self, Error> {
        try_pin_init!(Self {
            value: 42,
            // Mutex requires pinned initialization in place:
            lock <- kernel::new_mutex!(0, "MyData::lock"),
        })
    }
}
```

<details>

- Explain why `new_mutex!` uses `<--` syntax inside `try_pin_init!` to perform in-place field initialization.
- Emphasize that moving a struct after `mutex_init` in C causes undefined behavior, which `pin-init` prevents statically.

</details>
