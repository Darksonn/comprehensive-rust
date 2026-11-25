---
minutes: 10
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Multiple Allocators

```rust,ignore
use kernel::prelude::*;

// kmalloc: physically & virtually contiguous memory (best for small objects)
let small_buf = KBox::new([0u8; 256], GFP_KERNEL)?;

// vmalloc: virtually contiguous, physically non-contiguous (1 MiB table)
let large_table = VVec::from_elem(0u8, 1024 * 1024, GFP_KERNEL)?;

// kvmalloc: tries kmalloc first, falls back to vmalloc if contiguous memory is low
let flexible_buf = KVVec::from_elem(0u8, 64 * 1024, GFP_KERNEL)?;
```

## Comparing Kernel Allocators

| Allocator | Smart Pointer Type | Physical Layout | Best Used For |
| :--- | :--- | :--- | :--- |
| **`kmalloc`** | `KBox<T>`, `KVec<T>` | Physically & virtually contiguous | Small objects (≲ 32 KiB / a few pages), driver state structs |
| **`vmalloc`** | `VBox<T>`, `VVec<T>` | Virtually contiguous only | Very large memory buffers, dynamically sized tables |
| **`kvmalloc`** | `KVBox<T>`, `KVVec<T>` | Tries contiguous, falls back to virtual | General allocations that may vary from small to large sizes |

<details>

- For boxes, usually use `kmalloc` (`KBox<T>`). For vectors, usually use `kvmalloc` (`KVVec<T>` rather than `KVec<T>`), because vectors can grow large and `kvmalloc` falls back to `vmalloc` when contiguous memory is scarce.
- Explain why `kmalloc` can fail for large allocations due to physical memory fragmentation, and why `kvmalloc` is preferred for large buffers.
- Large arrays aren't boxed here: `VBox::new([0u8; N])` would first build the array on the 16 KiB kernel stack.

</details>
