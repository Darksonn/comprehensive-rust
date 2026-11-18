---
minutes: 10
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Mutex Guards

```rust,ignore
impl RustMiscDevice {
    fn set_value(&self, mut reader: UserSliceReader) -> Result<isize> {
        let new_value = reader.read::<i32>()?;
        let mut guard = self.inner.lock();

        dev_info!(self.dev, "-> Copying data from userspace (value: {})\n", new_value);

        guard.value = new_value;
        Ok(0)
    }
}
```

```c
static long set_value(struct rust_misc_device *dev, const char __user *buf)
{
    int new_value;

    if (copy_from_user(&new_value, buf, sizeof(new_value)))
        return -EFAULT;

    guard(mutex)(&dev->inner);
    dev_info(dev->dev, "-> Copying data from userspace (value: %d)\n", new_value);
    dev->value = new_value;

    return 0;
}
```

<details>

- **Acquiring the lock:** `self.inner.lock()` returns a `MutexGuard<'_, Inner>`.
- **No missing unlocks:** Like modern C's `guard(mutex)` (`<linux/cleanup.h>`), `MutexGuard` releases the lock automatically when it goes out of scope (including on `?` early returns).
- **Data encapsulation:** Unlike C's `guard(mutex)` (where the developer must still remember which fields require the lock), Rust's `Mutex<T>` makes it impossible to name `guard.value` unless the lock guard is held.
- Notice also that `self.dev` can be accessed freely without the guard because it is outside `Mutex<Inner>`.

</details>
