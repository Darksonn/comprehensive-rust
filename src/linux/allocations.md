---
minutes: 5
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Kernel Allocations

Kernel allocations differ from normal allocations in three important ways:

- Allocation failure.
- Allocator flags.
- Multiple allocators.

```rust,ignore
use kernel::prelude::*;

// Explicit allocation flags (e.g., GFP_KERNEL, GFP_ATOMIC):
let val: KBox<MyDriverData> = KBox::new(MyDriverData::new(), GFP_KERNEL)?;

let mut numbers: KVec<i32> = KVec::new();
numbers.push(42, GFP_KERNEL)?;
```

<details>

- Discuss why the kernel requires explicit allocation flags (sleeping vs. non-sleeping contexts).
- Emphasize that all kernel collections return `Result<_, AllocError>` and must use the `?` operator.

</details>
