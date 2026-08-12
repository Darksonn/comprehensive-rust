---
minutes: 15
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# The Misc Device Sample

The kernel sample [`samples/rust/rust_misc_device.rs`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/samples/rust/rust_misc_device.rs) brings together all the concepts we have covered today:

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
        KBox::try_pin_init(
            try_pin_init!(RustMiscDevice {
                inner <- new_mutex!(Inner {
                    value: 0_i32,
                    buffer: KVVec::new(),
                }),
                dev: dev,
            }),
            GFP_KERNEL,
        )
    }

    fn read(device: &RustMiscDevice, _file: &File, writer: &mut UserSliceWriter, offset: u64) -> Result<usize> {
        // Reads from device.inner buffer into user space...
    }

    fn write(device: &RustMiscDevice, _file: &File, reader: &mut UserSliceReader, offset: u64) -> Result<usize> {
        // Writes from user space into device.inner buffer...
    }
}
```

## The Limitation of the Sample

In this default sample, `open()` creates a **new `RustMiscDevice` instance on the heap for each file descriptor**:
- If Process A and Process B both open `/dev/rust_misc_device`, they each have their own isolated buffer.
- Data written by Process A is not visible to Process B.

<details>

- Walk through the anatomy of `MiscDevice`: `open()` returns `Self::Ptr` (`Pin<KBox<Self>>`), which the kernel stores in `file->private_data`.
- For subsequent `read`, `write`, and `ioctl` operations, the kernel passes `&RustMiscDevice` back to the driver methods.
- Point out that this per-file state isolation is fine for simple echo devices, but IPC drivers (like Binder) require state that is shared across processes.

</details>
