---
minutes: 10
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Mutexes vs. Spinlocks

The Linux kernel provides both sleeping locks (**`Mutex<T>`**) and non-sleeping locks (**`SpinLock<T>`**). Choosing between them depends on the execution context:

```rust,ignore
use kernel::sync::{Mutex, SpinLock};

// Sleeping lock: for process context
let mutex_state = Mutex::new(MyData::new());

// Spinning lock: for atomic/interrupt contexts
let spinlock_state = SpinLock::new(MyData::new());
```

## Key Differences

| Feature | `Mutex<T>` | `SpinLock<T>` |
| :--- | :--- | :--- |
| **Contention Behavior** | Puts thread to **sleep** until available | **Spins** (busy-waits) on the CPU |
| **Valid Contexts** | Process context only | Process, atomic, and interrupt contexts |
| **Operations Allowed Inside** | Can sleep (e.g., `GFP_KERNEL`, I/O) | **Cannot sleep** (no `GFP_KERNEL`, no blocking) |
| **Typical Hold Time** | Long (disk/network I/O, large copies) | Very short (updating small state/counters) |

## The Golden Rule

> [!WARNING]
> **Never sleep while holding a `SpinLock`!**
> Attempting to sleep or allocate with `GFP_KERNEL` while holding a `SpinLock` will trigger kernel lockdep warnings or system lockups. Use `GFP_ATOMIC` if allocation is required while holding a spinlock.

<details>

- Discuss why spinlocks disable preemption (and sometimes interrupts) on the local CPU core.
- Compare with user-space mutexes (which often spin briefly before sleeping via futex).

</details>
