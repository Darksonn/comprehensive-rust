---
minutes: 35
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Kernel Pointers

The Linux kernel cannot use standard library pointer types like `std::boxed::Box` or `std::sync::Arc` because kernel memory allocations can fail and require explicit allocation flags (`GFP_KERNEL`, `GFP_ATOMIC`).

Rust for Linux provides custom pointer types in `kernel::alloc` and `kernel::sync`:

- **`KBox<T>`:** Owned heap allocation using the `Kmalloc` slab allocator.
- **`KVBox<T>`:** Owned heap allocation using `KVmalloc`, which attempts physically contiguous allocation (`kmalloc`) and falls back to virtually contiguous allocation (`vmalloc`) for large sizes.
- **`Arc<T>`:** Thread-safe reference-counted pointer with explicit allocation flags.
- **`ARef<T>`:** Smart pointer for types reference-counted by C kernel code (implementing `AlwaysRefCounted`). This enables safe sharing of C structs like `task_struct` or `file` across Rust and C boundaries.

```rust,ignore
use kernel::prelude::*;
use kernel::alloc::{KBox, KVBox};

fn allocate_buffers() -> Result<()> {
    // Heap allocation requiring an explicit GFP_KERNEL flag:
    let small_buf: KBox<[u8; 64]> = KBox::new([0; 64], GFP_KERNEL)?;
    let large_buf: KVBox<[u8; 4096]> = KVBox::new([0; 4096], GFP_KERNEL)?;
    Ok(())
}
```

<details>

- Highlight the difference between `GFP_KERNEL` (can sleep to reclaim memory) and `GFP_ATOMIC` (cannot sleep, used in spinlocks and interrupts).
- Explain how `ARef<T>` allows Rust code to participate in C refcounting (`get_task_struct` / `put_task_struct`) via RAII.

</details>
