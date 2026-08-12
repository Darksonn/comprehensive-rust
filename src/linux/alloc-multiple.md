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

// kmalloc: physically & virtually contiguous memory (best for small objects & DMA)
let small_buf = KBox::new([0u8; 256], GFP_KERNEL)?;

// vmalloc: virtually contiguous, physically non-contiguous (for large allocations)
let large_table = VBox::new([0u8; 1024 * 1024], GFP_KERNEL)?;

// kvmalloc: tries kmalloc first, falls back to vmalloc if contiguous memory is low
let flexible_buf = KVBox::new([0u8; 64 * 1024], GFP_KERNEL)?;
```

## Comparing Kernel Allocators

| Allocator | Smart Pointer Type | Physical Layout | Best Used For |
| :--- | :--- | :--- | :--- |
| **`kmalloc`** | `KBox<T>`, `KVec<T>` | Physically & virtually contiguous | Small objects (< 128 KB), driver state structs, DMA buffers |
| **`vmalloc`** | `VBox<T>`, `VVec<T>` | Virtually contiguous only | Very large memory buffers, dynamically sized tables |
| **`kvmalloc`** | `KVBox<T>`, `KVVec<T>` | Tries contiguous, falls back to virtual | General allocations that may vary from small to large sizes |

<details>

- Explain why device drivers often require `kmalloc` (DMA hardware often requires contiguous physical memory addresses).
- Explain why `kmalloc` can fail for large allocations due to physical memory fragmentation, and why `kvmalloc` is preferred for large buffers.

</details>
