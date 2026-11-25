---
minutes: 10
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

To see this in action, use a single file descriptor for both the write and the read:

```c
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>

int main(void) {
    char buf[64];
    int a = open("/dev/rust-misc-device", O_RDWR);
    if (a < 0) { perror("open a"); return 1; }
    if (write(a, "hello\n", 6) != 6) { perror("write a"); return 1; }
    /* write_iter resets the file position to 0, so read straight back. */
    ssize_t n = read(a, buf, sizeof(buf));
    if (n < 0) { perror("read a"); return 1; }
    printf("fd a: read %zd bytes: %.*s", n, (int)n, buf);

    int b = open("/dev/rust-misc-device", O_RDWR);
    if (b < 0) { perror("open b"); return 1; }
    n = read(b, buf, sizeof(buf));
    if (n < 0) { perror("read b"); return 1; }
    printf("fd b: read %zd bytes\n", n);

    close(a);
    close(b);
    return 0;
}
```

```bash
gcc -static -o test_misc test_misc.c   # on the host, in ~/learn-rust
# Inside QEMU:
insmod /mnt/linux/samples/rust/rust_misc_device.ko
/mnt/test_misc
# fd a: read 6 bytes: hello
# fd b: read 0 bytes
```

<details>

- Walk through the two structs in `rust_misc_device.rs`:
  - `RustMiscDevice` (implementing `MiscDevice`) is stored inside `MiscDeviceRegistration<RustMiscDevice>` and passed to `open()` via `misc: &MiscDeviceRegistration<Self>` (which derefs to `&RustMiscDevice`).
  - `Self::File` (`Pin<KBox<RustMiscDeviceFile>>`) is returned by `open()` and stored in `file->private_data`.
- For subsequent `read_iter`, `write_iter`, and `ioctl` operations, the kernel passes `Self::File` (borrowed via `kiocb.file()`) back to the driver methods.
- In `read_iter()`, `iov.simple_read_from_buffer(kiocb.ki_pos_mut(), &inner.buffer)` reads starting at `*ki_pos` and advances `*ki_pos` (returning `0` at EOF once `*ki_pos` reaches `buffer.len()`).
- `echo hello > /dev/rust-misc-device` followed by `cat /dev/rust-misc-device` prints nothing: each command opens a fresh file descriptor with its own empty buffer. That is why we need a small C program that writes and reads on the same descriptor.
- Point out that per-file state isolation is fine for simple echo devices, but IPC drivers (like Binder) require state that is shared across processes.

</details>
