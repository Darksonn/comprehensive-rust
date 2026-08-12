---
minutes: 5
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

struct RustMiscDevice {
    inner: Mutex<Inner>,
    dev: ARef<Device>,
}
```

```c
struct rust_misc_device {
    int value;
    char *buffer;
    size_t buffer_len;
    size_t buffer_cap;
    struct mutex inner;
    struct device *dev;
};
```

<details>

- Emphasize that a Rust mutex is a container for the data.

</details>
