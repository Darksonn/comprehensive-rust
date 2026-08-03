---
minutes: 35
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Synchronization

On Day 1 Morning, we started with a static `miscdevice` driver that returned read-only greeting bytes. To store **mutable state** (such as a buffer or counter updated across user-space `read`, `write`, and `ioctl` calls, as in `samples/rust/rust_misc_device.rs`), we must protect that state from concurrent access using kernel synchronization primitives.

The Linux kernel is a concurrent environment with strict rules regarding atomic and sleeping contexts. Rust for Linux integrates locking primitives with Rust's type system and `pin-init`:

- **Sleeping Locks (`Mutex<T>`):** Can sleep while waiting for acquisition. Must only be used in process context where sleeping is permitted.
- **Atomic Locks (`SpinLock<T>`):** Never sleep and disable preemption while held. Safe for atomic contexts and interrupt handlers.
- **Pinned Initialization Requirement:** Because kernel locks wrap C structures that register lockdep metadata and list heads, every lock must be initialized in place using `pin-init` macros (`new_mutex!`, `new_spinlock!`).

```rust,ignore
use kernel::prelude::*;
use kernel::sync::{SpinLock, SpinLockGuard};

struct SharedState {
    counter: SpinLock<u64>,
}

impl SharedState {
    pub fn new() -> impl PinInit<Self, Error> {
        try_pin_init!(Self {
            counter <- kernel::new_spinlock!(0, "SharedState::counter"),
        })
    }

    pub fn increment(&self) {
        // Locking returns an RAII guard that releases the spinlock on drop:
        let mut guard: SpinLockGuard<'_, u64> = self.counter.lock();
        *guard += 1;
    }
}
```

<details>

- Discuss how `lockdep` (Linux Lock Validator) names are provided via `"SharedState::counter"` in the initialization macro.
- Contrast kernel `Mutex` with standard library `Mutex`: holding a kernel spinlock prevents sleeping, so allocations inside a spinlock critical section must use `GFP_ATOMIC`.

</details>
