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
use kernel::task::Task;
use kernel::uaccess::{UserSliceReader, UserSliceWriter};
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

struct FileContext {
    shared: Arc<SharedState>,
}

#[vtable]
impl MiscDevice for FileContext {
    type Ptr = KBox<Self>;

    fn open(_file: &File, misc: &MiscDeviceRegistration<Self>) -> Result<KBox<Self>> {
        // Retrieve the shared state reference from the registration or static context:
        let shared = Arc::clone(misc.as_ref());
        KBox::new(FileContext { shared }, GFP_KERNEL)
    }

    fn write(context: &FileContext, _file: &File, reader: &mut UserSliceReader, _offset: u64) -> Result<usize> {
        let len = reader.len();
        let mut data = KVec::new();
        data.resize(len, 0, GFP_KERNEL)?;
        reader.read_slice(&mut data)?;

        let pid = Task::current().pid();

        let mut guard = context.shared.inner.lock();
        // Clear previous message and format new message with PID prefix:
        guard.buffer.clear();
        use core::fmt::Write;
        let _ = write!(guard.buffer, "[PID {}]: ", pid);
        guard.buffer.extend_from_slice(&data, GFP_KERNEL)?;

        Ok(len)
    }

    fn read(context: &FileContext, _file: &File, writer: &mut UserSliceWriter, offset: u64) -> Result<usize> {
        let guard = context.shared.inner.lock();
        let buf_len = guard.buffer.len();

        if offset as usize >= buf_len {
            return Ok(0); // EOF
        }

        let slice = &guard.buffer[offset as usize..];
        let to_copy = core::cmp::min(writer.len(), slice.len());

        // Clone slice to release the mutex before interacting with user space:
        let mut temp = KVec::new();
        temp.extend_from_slice(&slice[..to_copy], GFP_KERNEL)?;
        drop(guard);

        writer.write_slice(&temp)?;
        Ok(to_copy)
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
            name: c_str!("rust_ipc"),
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
