---
minutes: 10
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# What Went Wrong in `rust/kernel/mm/virt.rs`?

[`rust/kernel/mm/virt.rs`](https://github.com/Darksonn/linux/blob/rfl-course-edu/rust/kernel/mm/virt.rs):

```rust,ignore
impl MmapReadGuard<'_> {
    pub fn vma_lookup(&self, vma_addr: usize) -> Option<&VmaRef> { ... }
}

impl VmaRef {
    pub fn zap_vma_range(&self, address: usize, size: usize) { ... }

    pub fn as_mixedmap_vma(&self) -> Option<&VmaMixedMap> {
        if self.flags() & flags::MIXEDMAP != 0 {
            Some(unsafe { VmaMixedMap::from_raw(self.as_ptr()) })
        } else {
            None
        }
    }
}

impl VmaMixedMap {
    pub fn vm_insert_page(&self, address: usize, page: &Page) -> Result { ... }
}
```

<details>

- Look at the types in [`rust/kernel/mm/virt.rs`](https://github.com/Darksonn/linux/blob/rfl-course-edu/rust/kernel/mm/virt.rs):
  - `vma_lookup(vma_addr)` (and `lock_vma_under_rcu(vma_addr)`) can return **any** VMA in the process's address space—anonymous memory, a file mapping, or another driver's VMA.
  - Yet `VmaRef` exposes `zap_vma_range` and `as_mixedmap_vma` $\to$ `vm_insert_page` as **safe** methods on *any* `&VmaRef`!
- Why is this a soundness / encapsulation flaw in `virt.rs`?
  - Holding the mmap read lock and checking `VM_MIXEDMAP` is **not** enough to allow arbitrary code to insert pages into or zap pages from a foreign VMA.
  - Another subsystem's `VM_MIXEDMAP` VMA may rely on invariants about which pages are mapped in its VMA.
  - Safe code should only be allowed to modify a VMA after proving that **this driver actually owns the VMA**.

</details>
