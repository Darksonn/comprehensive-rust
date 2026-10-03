---
minutes: 5
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Mutexes vs. Spinlocks

```rust,ignore
use kernel::sync::{Mutex, SpinLock};

// Sleeping lock (process context):
let mutex_state = new_mutex!(MyData::new());

// Spinning lock (atomic / interrupt context):
let spinlock_state = new_spinlock!(MyData::new());
```

| Feature | `Mutex<T>` | `SpinLock<T>` |
| :--- | :--- | :--- |
| **Contention** | Sleeps | Spins on CPU |
| **Valid Contexts** | Process context only | Process, atomic, and IRQ contexts |
| **While Held** | May sleep (`GFP_KERNEL`, I/O) | **Never sleep** (`GFP_ATOMIC` only) |

<details>

- Both `Mutex<T>` and `SpinLock<T>` use the exact same container pattern: data lives inside the lock and is accessed through a guard.
- **The Golden Rule:** Never sleep while holding a `SpinLock`! Allocating with `GFP_KERNEL` or waiting on I/O while holding a `SpinLock` triggers lockdep warnings or deadlocks.

</details>
