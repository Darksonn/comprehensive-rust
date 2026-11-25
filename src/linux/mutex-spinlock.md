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
| **Valid Contexts** | Process context only | Process & atomic context (not shared with IRQ handlers) |
| **While Held** | May sleep (`GFP_KERNEL`, I/O) | **Never sleep** (`GFP_ATOMIC` only) |

Locks shared with an interrupt handler need `SpinLockIrq<T>` (`new_spinlock_irq!`), which disables local interrupts while held.

<details>

- Both `Mutex<T>` and `SpinLock<T>` use the exact same container pattern: data lives inside the lock and is accessed through a guard.
- **The Golden Rule:** Never sleep while holding a `SpinLock`! Allocating with `GFP_KERNEL` or waiting on I/O while holding a `SpinLock` triggers `BUG: sleeping function called from invalid context` (with `CONFIG_DEBUG_ATOMIC_SLEEP`) or deadlocks.
- `SpinLock` uses plain `spin_lock()`: if an IRQ handler on the same CPU tries to take a lock that the interrupted code holds, it deadlocks. `SpinLockIrq` uses `spin_lock_irq_disable()`; this matters for Day 2's Interrupts section.
- `new_mutex!` returns an in-place initializer; we'll see why in Pinned Initialization.

</details>
