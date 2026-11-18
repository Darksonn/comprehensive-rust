---
minutes: 15
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# The Misc Device Sample

The kernel sample [`samples/rust/rust_misc_device.rs`](https://github.com/Darksonn/linux/blob/rfl-course-edu/samples/rust/rust_misc_device.rs) brings together all the concepts we have covered today:

```rust,ignore
struct Inner {
    value: i32,
    buffer: KVVec<u8>,
}

/// Data shared by all open files for the same instance of this miscdevice.
#[pin_data]
struct RustMiscDevice {}

/// Data that is stored per file for this miscdevice.
#[pin_data(PinnedDrop)]
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

        KBox::try_pin_init(
            try_pin_init!(RustMiscDeviceFile {
                inner <- new_mutex!(Inner {
                    value: 0_i32,
                    buffer: KVVec::new(),
                }),
                dev: dev,
            }),
            GFP_KERNEL,
        )
    }

    fn read_iter(mut kiocb: Kiocb<'_, Self::File>, iov: &mut IovIterDest<'_>) -> Result<usize> {
        // Reads from kiocb.file().inner buffer into user space...
    }

    fn write_iter(mut kiocb: Kiocb<'_, Self::File>, iov: &mut IovIterSource<'_>) -> Result<usize> {
        // Writes from user space into kiocb.file().inner buffer...
    }
}
```

## The Limitation of the Sample

In this default sample, `RustMiscDevice` is empty and `open()` allocates a **new `RustMiscDeviceFile` buffer on the heap for each file descriptor**:
- If Process A and Process B both open `/dev/rust-misc-device`, they each have their own isolated buffer.
- Data written by Process A is not visible to Process B.

<details>

- Walk through the two structs in `rust_misc_device.rs`:
  - `RustMiscDevice` (implementing `MiscDevice`) is stored inside `MiscDeviceRegistration<RustMiscDevice>` and passed to `open()` via `misc: &MiscDeviceRegistration<Self>` (which derefs to `&RustMiscDevice`).
  - `Self::File` (`Pin<KBox<RustMiscDeviceFile>>`) is returned by `open()` and stored in `file->private_data`.
- For subsequent `read_iter`, `write_iter`, and `ioctl` operations, the kernel passes `Self::File` (borrowed via `kiocb.file()`) back to the driver methods.
- Point out that per-file state isolation is fine for simple echo devices, but IPC drivers (like Binder) require state that is shared across processes.

</details>
