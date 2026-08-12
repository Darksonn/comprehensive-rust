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

#[pin_data]
struct RustMiscDevice {
    #[pin]
    inner: Mutex<Inner>,
    dev: ARef<Device>,
}

#[vtable]
impl MiscDevice for RustMiscDevice {
    type Ptr = Pin<KBox<Self>>;

    fn open(_file: &File, misc: &MiscDeviceRegistration<Self>) -> Result<Pin<KBox<Self>>> {
        let dev = ARef::from(misc.device());

        dev_info!(dev, "Opening Rust Misc Device Sample\n");

        KBox::try_pin_init(
            try_pin_init! {
                RustMiscDevice {
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

</details>
