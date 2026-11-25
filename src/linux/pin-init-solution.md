---
minutes: 10
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# In-Place Initialization with pin-init

To initialize kernel types containing locks or self-referential pointers safely without moving them through the stack, Rust for Linux provides **in-place pinned initialization**:

```rust,ignore
struct Inner {
    value: i32,
    buffer: KVVec<u8>,
}

/// Data shared by all open files for the same miscdevice.
#[pin_data]
struct RustMiscDevice {}

/// Data stored per open file.
#[pin_data]
struct RustMiscDeviceFile {
    #[pin]
    inner: Mutex<Inner>,
    dev: ARef<Device>,
}

#[vtable]
impl MiscDevice for RustMiscDevice {
    type File = Pin<KBox<RustMiscDeviceFile>>;

    fn open(_file: &File, misc: &MiscDeviceRegistration<Self>) -> Result<Self::File> {
        let dev = ARef::from(misc.device());

        dev_info!(dev, "Opening Rust Misc Device Sample\n");

        KBox::try_pin_init(
            try_pin_init! {
                RustMiscDeviceFile {
                    inner <- new_mutex!(Inner {
                        value: 0_i32,
                        buffer: KVVec::new(),
                    }),
                    dev: dev,
                }
            },
            GFP_KERNEL,
        )
    }
}
```

<details>

- Highlight how `<-` distinguishes in-place initialization from normal field assignment (`:`).
- Explain importance of `#[pin]` and `#[pin_data]` in struct initialization.
- `open` returns `Self::File` (here `Pin<KBox<RustMiscDeviceFile>>`): each open file gets its own pinned, heap-allocated `RustMiscDeviceFile`, while `RustMiscDevice` holds data shared by all open files (empty here).
- `KBox::try_pin_init` allocates first and then runs the initializer directly in the heap memory, so the `Mutex` is never moved after `mutex_init`.

</details>
