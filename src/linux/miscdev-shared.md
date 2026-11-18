---
minutes: 15
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Sharing State Across Open Files

To share state across every process that opens the device, we store an `Arc<SharedState>` inside the registration struct and clone it in `open()`:

```rust,ignore
#[pin_data]
struct SharedState {
    #[pin]
    inner: Mutex<Inner>,
}

/// Data shared by all open files for the same instance of this miscdevice.
#[pin_data]
struct RustMessageBoard {
    shared: Arc<SharedState>,
}

/// Data that is stored per file for this miscdevice.
#[pin_data(PinnedDrop)]
struct RustMessageBoardFile {
    shared: Arc<SharedState>,
    dev: ARef<Device>,
}

#[vtable]
impl MiscDevice for RustMessageBoard {
    type File = Pin<KBox<RustMessageBoardFile>>;

    fn open(_file: &File, misc: &MiscDeviceRegistration<Self>) -> Result<Self::File> {
        let dev = ARef::from(misc.device());

        KBox::try_pin_init(
            try_pin_init!(RustMessageBoardFile {
                shared: Arc::clone(&misc.shared),
                dev,
            }),
            GFP_KERNEL,
        )
    }
    // ...
}
```

<details>

- **Live-code / walk through this transformation together on [`samples/rust/rust_misc_device.rs`](https://github.com/Darksonn/linux/blob/rfl-course-edu/samples/rust/rust_misc_device.rs):**
  1. In `init()`, allocate `let shared = Arc::pin_init(try_pin_init!(SharedState { ... }), GFP_KERNEL)?;` and pass `try_pin_init!(RustMiscDevice { shared })` to `MiscDeviceRegistration::register`.
  2. Why `Arc<SharedState>` instead of borrowing `&RustMessageBoard` directly in `RustMessageBoardFile`? Because `MiscDevice::File` must be `'static`: unregistering a `miscdevice` stops *new* `open()` calls, while already-open file descriptors remain valid until user space closes them.
  3. Point out that this completed shared-buffer starting point is also available in the repository at [`samples/rust/rust_message_board.rs`](https://github.com/Darksonn/linux/blob/rfl-course-edu/samples/rust/rust_message_board.rs) (exposing `/dev/rust-message-board`).
- **Lead-in to the exercise:** Right now `Inner` still holds a single `buffer: KVVec<u8>` that every `write_iter` clears and overwrites. Next, you'll turn it into a real multi-message board where readers stream messages one by one!

</details>
