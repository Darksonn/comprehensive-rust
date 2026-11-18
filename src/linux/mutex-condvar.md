---
minutes: 5
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Condition Variables

`CondVar` wraps the kernel's `struct wait_queue_head` so tasks can sleep until a condition protected by a lock becomes true:

```rust,ignore
use kernel::sync::{new_condvar, new_mutex, CondVar, Mutex};

#[pin_data]
struct Channel {
    #[pin]
    ready: Mutex<bool>,
    #[pin]
    condvar: CondVar,
}

fn wait_until_ready(ch: &Channel) -> Result {
    let mut guard = ch.ready.lock();
    while !*guard {
        if ch.condvar.wait_interruptible(&mut guard) {
            return Err(ERESTARTSYS);
        }
    }
    Ok(())
}

fn mark_ready(ch: &Channel) {
    *ch.ready.lock() = true;
    ch.condvar.notify_all();
}
```

<details>

- **Compile-time lock proof:** `wait_interruptible(&mut guard)` requires a mutable reference to an active lock `Guard`. It atomically unlocks the mutex, puts the task to sleep, and reacquires the mutex before returning—making lost-wakeup races from checking the condition unlocked impossible.
- **Why a `while` loop?** Waiters can wake up spuriously or compete with other woken threads, so always re-check the condition inside `while`.
- **Handling signals (`#[must_use]`):** `wait_interruptible` returns `true` when a signal (such as <kbd>Ctrl</kbd> + <kbd>C</kbd>) is pending; returning `Err(ERESTARTSYS)` lets the VFS abort or restart the syscall cleanly instead of hanging in an unkillable (`D`) state.
- **`notify_one()` vs. `notify_all()`:** Use `notify_one()` when one consumer claims a work item; use `notify_all()` when broadcasting state changes that all waiting readers should observe.

</details>
