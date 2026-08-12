---
minutes: 10
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Releasing Guards Early

Holding a lock for longer than necessary increases contention. In particular, copying data to user space can fault or take time, so we should release the lock before performing user copies whenever possible.

You can explicitly release a lock early using **`drop(guard)`**:

```rust,ignore
impl RustMiscDevice {
    fn get_value(&self, mut writer: UserSliceWriter) -> Result<isize> {
        let guard = self.inner.lock();
        let value = guard.value;

        // Free-up the lock and use our locally cached instance from here
        drop(guard);

        dev_info!(
            self.dev,
            "-> Copying data to userspace (value: {})\n",
            &value
        );

        writer.write::<i32>(&value)?;
        Ok(0)
    }
}
```

## Why Early Drop Matters

- **Minimizing Critical Sections:** Once `value` is copied to a local variable on the stack, the protected state is no longer needed.
- **Avoiding Lock Contention During User Copies:** `UserSliceWriter::write` copies memory to user space, which may page fault or sleep. Releasing the lock first ensures other threads aren't blocked waiting for the copy to finish.
- **Compile-Time Safety:** After calling `drop(guard)`, the compiler prevents any further access to fields through `guard`.

<details>

- Point out that attempting to use `guard` after `drop(guard)` triggers a compile error (`use of moved value: guard`).
- Contrast with C: In C, if a developer writes `mutex_unlock(&dev->inner)` and then forgets and accesses `dev->value` in `copy_to_user`, the compiler cannot catch the data race. In Rust, you must copy out the data before dropping the guard.

</details>
