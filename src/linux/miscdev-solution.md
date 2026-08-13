---
minutes: 15
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Solution: Shared IPC Message Board

Here is a complete solution implementing a shared IPC message board driver using `Arc<SharedState>`:

```rust,ignore
// SPDX-License-Identifier: GPL-2.0

//! Shared IPC Message Board Driver.

use kernel::prelude::*;
use kernel::sync::{new_mutex, Arc, Mutex};
use kernel::fs::{File, Kiocb};
use kernel::iov::{IovIterDest, IovIterSource};
use kernel::miscdevice::{MiscDevice, MiscDeviceOptions, MiscDeviceRegistration};

module! {
    type: IpcModule,
    name: "rust_ipc",
    authors: ["Rust for Linux Course"],
    description: "Shared IPC Message Board Sample",
    license: "GPL",
}

struct Inner {
    buffer: KVVec<u8>,
}

#[pin_data]
struct SharedState {
    #[pin]
    inner: Mutex<Inner>,
}

#[pin_data]
struct FileContext {
    shared: Arc<SharedState>,
}

#[vtable]
impl MiscDevice for FileContext {
    type Data = Arc<SharedState>;
    type Ptr = Pin<KBox<Self>>;

    fn open(_file: &File, misc: &MiscDeviceRegistration<Self>) -> Result<Pin<KBox<Self>>> {
        let shared = Arc::clone(misc.data());
        KBox::try_pin_init(
            try_pin_init!(FileContext { shared }),
            GFP_KERNEL,
        )
    }

    fn write_iter(mut kiocb: Kiocb<'_, Self::Ptr>, iov: &mut IovIterSource<'_>) -> Result<usize> {
        let me = kiocb.file();
        let mut data = KVec::new();
        iov.copy_from_iter_vec(&mut data, GFP_KERNEL)?;

        let pid = current!().pid();

        // Format prefix onto stack
        let mut buf = [0u8; 32];
        let mut formatter = kernel::str::Formatter::new(&mut buf);
        use core::fmt::Write;
        let _ = write!(formatter, "[PID {}]: ", pid);
        let prefix_len = formatter.bytes_written();
        let prefix = &buf[..prefix_len];

        let mut guard = me.shared.inner.lock();
        // Clear previous message and format new message with PID prefix:
        guard.buffer.clear();
        guard.buffer.extend_from_slice(prefix, GFP_KERNEL)?;
        guard.buffer.extend_from_slice(&data, GFP_KERNEL)?;

        // Reset position on write
        *kiocb.ki_pos_mut() = 0;

        Ok(data.len())
    }

    fn read_iter(mut kiocb: Kiocb<'_, Self::Ptr>, iov: &mut IovIterDest<'_>) -> Result<usize> {
        let me = kiocb.file();
        let guard = me.shared.inner.lock();
        let buf_len = guard.buffer.len();

        let offset = kiocb.ki_pos();
        if offset as usize >= buf_len {
            return Ok(0); // EOF
        }

        let slice = &guard.buffer[offset as usize..];
        let to_copy = core::cmp::min(iov.len(), slice.len());

        // Clone slice to release the mutex before interacting with user space:
        let mut temp = KVec::new();
        temp.extend_from_slice(&slice[..to_copy], GFP_KERNEL)?;
        drop(guard);

        let num_written = iov.copy_to_iter(&temp);
        *kiocb.ki_pos_mut() += num_written as i64;

        Ok(num_written)
    }
}

struct IpcModule {
    _reg: Pin<KBox<MiscDeviceRegistration<FileContext>>>,
}

impl kernel::Module for IpcModule {
    fn init(_module: &'static ThisModule) -> Result<Self> {
        pr_info!("Rust IPC Message Board loaded\n");

        let shared = Arc::pin_init(
            try_pin_init!(SharedState {
                inner <- new_mutex!(Inner {
                    buffer: KVVec::new(),
                }),
            }),
            GFP_KERNEL,
        )?;

        let options = MiscDeviceOptions {
            name: c"rust_ipc",
            parent: None,
        };

        let reg = KBox::try_pin_init(
            MiscDeviceRegistration::register(options, shared),
            GFP_KERNEL,
        )?;

        Ok(IpcModule { _reg: reg })
    }
}
```

<details>

- **Architectural Pattern:** This pattern—separating global driver state (`SharedState`) from per-open file state (`FileContext`) via `Arc<T>`—is the exact architectural foundation used by production kernel IPC drivers like **Binder**.
- **Early Lock Release:** Highlight how `read()` copies data into a small local buffer and invokes `drop(guard)` before `writer.write_slice(&temp)?` touches user space memory.
- **OOM Safety:** Point out that all buffer resizes, clones, and allocations use `GFP_KERNEL` and the `?` operator.

</details>
