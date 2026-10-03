---
minutes: 25
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Live Coding: Introducing `DriverVma`

Current workaround in [`drivers/android/binder/page_range.rs`](https://github.com/Darksonn/linux/blob/rfl-course-edu/drivers/android/binder/page_range.rs):

```rust,ignore
static BINDER_VM_OPS: AssertSync<bindings::vm_operations_struct> =
    AssertSync(pin_init::zeroed());

fn check_vma(vma: &virt::VmaRef, owner: *const ShrinkablePageRange) -> Option<&virt::VmaMixedMap> {
    let vm_ops = unsafe { (*vma.as_ptr()).vm_ops };
    if !ptr::eq(vm_ops, &BINDER_VM_OPS.0) {
        return None;
    }
    let vm_private_data = unsafe { (*vma.as_ptr()).vm_private_data };
    if !ptr::eq(vm_private_data, owner.cast()) {
        return None;
    }
    vma.as_mixedmap_vma()
}
```

- **Goal:** Move VMA ownership into the type system ([`rust/kernel/mm/virt.rs`](https://github.com/Darksonn/linux/blob/rfl-course-edu/rust/kernel/mm/virt.rs)) with `DriverVma`.

<details>

- Show the emergency CVE fix in [`drivers/android/binder/page_range.rs`](https://github.com/Darksonn/linux/blob/rfl-course-edu/drivers/android/binder/page_range.rs):
  - During `register_with_vma(vma: &VmaNew)`, Binder sets `vma->vm_ops = &BINDER_VM_OPS` and `vma->vm_private_data = self` using `unsafe`.
  - After `vma_lookup` / `lock_vma_under_rcu`, `check_vma` checks `vm_ops` and `vm_private_data` before calling `as_mixedmap_vma()` or `zap_vma_range()`.
- Why we want to fix this in [`rust/kernel/mm/virt.rs`](https://github.com/Darksonn/linux/blob/rfl-course-edu/rust/kernel/mm/virt.rs):
  1. Right now, `VmaRef::zap_vma_range` and `VmaRef::as_mixedmap_vma` are still callable on an unchecked `VmaRef`—nothing in the type system forces a driver to call `check_vma`!
  2. Touching `vm_ops` and `vm_private_data` via raw pointers in driver code shouldn't be necessary.
- **Switch to editor (`~/linux/rust/kernel/mm/virt.rs` and `drivers/android/binder/page_range.rs`):**
  - Introduce `DriverVma` representing a VMA known to be owned by the current driver.
  - Move `zap_vma_range` and `as_mixedmap_vma` off of arbitrary `VmaRef` and onto `DriverVma`.
  - Provide a safe way to tag a `VmaNew` during `mmap` and down-cast a looked-up `VmaRef` to `DriverVma`.

</details>
