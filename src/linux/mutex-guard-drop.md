---
minutes: 10
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Releasing Guards Early

```rust,ignore
// Option 1: Explicit `drop`
fn get_value(&self, mut writer: UserSliceWriter) -> Result<isize> {
    let guard = self.inner.lock();
    let value = guard.value;
    drop(guard);

    writer.write::<i32>(&value)?;
    Ok(0)
}

// Option 2: Block scope
fn get_value_scoped(&self, mut writer: UserSliceWriter) -> Result<isize> {
    let value = {
        let guard = self.inner.lock();
        guard.value
    };

    writer.write::<i32>(&value)?;
    Ok(0)
}
```

<details>

- **Minimizing critical sections:** Copying data to user space (`writer.write`) can page fault or sleep, so we want to release the lock first.
- **Two ways to release a guard early:**
  1. **Explicit unlock (`drop(guard)`):** Consumes the guard by value and unlocks the mutex immediately.
  2. **Lexical block scope (`{ ... }`):** The guard is dropped automatically at the closing brace of the inner block, returning just the copied `value`.
- **Compile-time safety:** After `drop(guard)` or the closing brace, attempting to access `guard.value` is a compile error. In C, if you call `mutex_unlock(&dev->inner)` and accidentally pass `dev->value` to `copy_to_user`, the compiler won't catch the race.

</details>
