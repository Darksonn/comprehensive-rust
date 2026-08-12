---
minutes: 10
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Mutex Guards

Accessing the data inside a `Mutex<T>` requires acquiring the lock, which returns a **`MutexGuard<'_, T>`**:

```rust,ignore
impl RustMiscDevice {
    fn set_value(&self, mut reader: UserSliceReader) -> Result<isize> {
        let new_value = reader.read::<i32>()?;
        let mut guard = self.inner.lock();

        dev_info!(
            self.dev,
            "-> Copying data from userspace (value: {})\n",
            new_value
        );

        guard.value = new_value;
        Ok(0)
    }
}
```

## Comparison with C

**Traditional C (`mutex_lock` / `mutex_unlock`):**
```c
static long set_value(struct rust_misc_device *dev, const char __user *buf)
{
    int new_value;

    if (copy_from_user(&new_value, buf, sizeof(new_value)))
        return -EFAULT;

    mutex_lock(&dev->inner);
    dev_info(dev->dev, "-> Copying data from userspace (value: %d)\n", new_value);
    dev->value = new_value;
    mutex_unlock(&dev->inner);

    return 0;
}
```

**Modern C with `guard(mutex)` (`<linux/cleanup.h>`):**
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

- **No Missing Unlocks:** Both Rust's `MutexGuard` and C's `guard(mutex)` release the lock automatically on scope exit (including early returns and error paths).
- **Data Encapsulation:** Unlike C's `guard(mutex)` (where the developer must still remember which fields require the lock), Rust's `Mutex<T>` prevents accessing the protected fields entirely unless the guard is held.

</details>
