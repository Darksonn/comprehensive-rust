---
minutes: 10
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Allocator Flags

```rust,ignore
use kernel::prelude::*;

// In process context (can sleep to reclaim memory or wait for I/O):
let data = KBox::new(MyDriverState::new(), GFP_KERNEL)?;

// In atomic / interrupt context or while holding a spinlock (cannot sleep):
let _guard = spinlock.lock();
let packet = KBox::new(Packet::new(), GFP_ATOMIC)?;
```

<details>

- Emphasize the golden rule of Linux kernel programming: **Never sleep in atomic context.**
- Using `GFP_KERNEL` inside an interrupt handler or while holding a spinlock triggers kernel warnings/panics (or lockdep/might_sleep assertions).

</details>
