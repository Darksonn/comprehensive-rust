---
minutes: 10
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Mutexes & Synchronization

Rust mutexes are *containers*!

```rust,ignore
struct Inner {
    value: i32,
    buffer: KVVec<u8>,
}

struct RustMiscDeviceFile {
    inner: Mutex<Inner>,
    dev: ARef<Device>,
}
```

Equivalent code in C:

```c
struct rust_misc_device_file {
    struct mutex inner;
    /* Protected by `inner`: */
    int value;
    char *buffer;
    size_t buffer_len;
    size_t buffer_cap;

    /* Not protected by `inner`: */
    struct device *dev;
};
```

<details>

- **Data inside vs. outside the container:** `value` and `buffer` live *inside* `Mutex<Inner>`, while `dev: ARef<Device>` sits *outside* the `Mutex` because logging with `dev_info!(self.dev, ...)` does not require holding the lock.
- **Contrast with C:** In C, `struct mutex inner` sits alongside the fields it protects. Nothing in the C type system stops code from reading or writing `dev->value` without locking `dev->inner`.
- **What if you forget to use a `Mutex` at all?** In Rust, concurrent driver callbacks (such as `ioctl`, `read_iter`, or `write_iter`) receive a **shared reference (`&self`)** because multiple CPUs can call them simultaneously. You cannot mutate fields through `&self` (`&mut self.buffer` fails to compile)—the compiler forces you to use a synchronization container like `Mutex<T>` or `SpinLock<T>` to obtain mutable access (`&mut Inner`) from `&self`.

</details>
