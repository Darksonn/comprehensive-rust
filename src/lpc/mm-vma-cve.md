---
minutes: 10
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Case Study: A Bug in `kernel::mm`

[`drivers/android/binder/page_range.rs`](https://github.com/Darksonn/linux/blob/rfl-course-edu/drivers/android/binder/page_range.rs):

```rust,ignore
// 1. During f_ops->mmap(vma: &VmaNew):
inner.vma_addr = vma.start();
vma.try_clear_maywrite()?;
vma.set_mixedmap();

// 2. Later, when inserting a page into the process's address space:
mm.mmap_read_lock()
    .vma_lookup(vma_addr)
    .ok_or(ESRCH)?
    .as_mixedmap_vma()
    .ok_or(ESRCH)?
    .vm_insert_page(user_page_addr, &new_page)?;

// 3. When reclaiming unused pages in the shrinker:
if let Some(vma) = mmap_read.vma_lookup(vma_addr) {
    vma.zap_vma_range(user_page_addr, PAGE_SIZE);
}
```

- Notice: **100% safe Rust** in the driver.
- **Question:** What can go wrong here?

<details>

- Recall our core rule from earlier: *If safe driver code causes a memory/MM bug, the bug is in the `kernel` crate abstraction.*
- Walk through what Rust Binder does:
  1. When userspace `mmap`s `/dev/binder`, Binder clears `VM_MAYWRITE` (so the mapping is read-only to userspace), sets `VM_MIXEDMAP`, and saves `vma_addr = vma.start()`.
  2. Later, when a transaction arrives or the shrinker runs, Binder locks the `mm`, looks up `vma_lookup(vma_addr)`, and calls `vm_insert_page` or `zap_vma_range`.
- Ask the MM folks in the room: **What happens if userspace calls `munmap()` on `vma_addr` and maps a different VMA at that same address?**
- Answer (`CVE-2026-43434`, reported by Jann Horn):
  - Userspace replaces the read-only Binder VMA with a **writable** VMA at `vma_addr`.
  - `vma_lookup(vma_addr)` returns the new VMA, and Binder inserts its supposedly read-only page into a writable VMA (or zaps pages from an unrelated VMA)!

</details>
