---
minutes: 5
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Solution: Clearing the Board (`O_TRUNC`)

```rust,ignore
// SPDX-License-Identifier: GPL-2.0

//! Rust message board character device solution.

use kernel::{
    device::Device,
    fs::{file::flags, File, Kiocb},
    iov::{IovIterDest, IovIterSource},
    miscdevice::{MiscDevice, MiscDeviceOptions, MiscDeviceRegistration},
    new_condvar, new_mutex,
    prelude::*,
    sync::{aref::ARef, Arc, CondVar, Mutex},
};

module! {
    type: RustMessageBoardModule,
    name: "rust_message_board",
    authors: ["Rust for Linux Course"],
    description: "Rust message board exercise",
    license: "GPL",
}

#[pin_data]
struct RustMessageBoardModule {
    #[pin]
    _miscdev: MiscDeviceRegistration<RustMessageBoard>,
}

impl kernel::InPlaceModule for RustMessageBoardModule {
    fn init(_module: &'static ThisModule) -> impl PinInit<Self, Error> {
        pin_init::pin_init_scope(move || {
            pr_info!("Initializing Rust Message Board\n");

            let shared = Arc::pin_init(
                try_pin_init!(SharedState {
                    inner <- new_mutex!(Inner {
                        base: 0,
                        messages: KVec::new(),
                    }),
                    condvar <- new_condvar!(),
                }),
                GFP_KERNEL,
            )?;

            let options = MiscDeviceOptions {
                name: c"rust-message-board",
            };

            Ok(try_pin_init!(Self {
                // SAFETY: `_miscdev` is stored in `RustMessageBoardModule` and dropped when the
                // module is unloaded; `mem::forget()` is never called on it.
                _miscdev <- unsafe {
                    MiscDeviceRegistration::register(
                        options,
                        try_pin_init!(RustMessageBoard { shared }),
                    )
                },
            }))
        })
    }
}

struct Inner {
    base: usize,
    messages: KVec<KVec<u8>>,
}

#[pin_data]
struct SharedState {
    #[pin]
    inner: Mutex<Inner>,
    #[pin]
    condvar: CondVar,
}

#[pin_data]
struct RustMessageBoard {
    shared: Arc<SharedState>,
}

#[pin_data(PinnedDrop)]
struct RustMessageBoardFile {
    shared: Arc<SharedState>,
    dev: ARef<Device>,
    #[pin]
    cursor: Mutex<usize>,
    nonblock: bool,
}

#[vtable]
impl MiscDevice for RustMessageBoard {
    type File = Pin<KBox<RustMessageBoardFile>>;

    fn open(file: &File, misc: &MiscDeviceRegistration<Self>) -> Result<Self::File> {
        let dev = ARef::from(misc.device());
        let nonblock = (file.flags() & flags::O_NONBLOCK) != 0;

        if (file.flags() & flags::O_TRUNC) != 0 {
            let mut inner = misc.shared.inner.lock();
            inner.base += inner.messages.len();
            inner.messages.clear();
        }

        KBox::try_pin_init(
            try_pin_init!(RustMessageBoardFile {
                shared: Arc::clone(&misc.shared),
                dev,
                cursor <- new_mutex!(0),
                nonblock,
            }),
            GFP_KERNEL,
        )
    }

    fn read_iter(kiocb: Kiocb<'_, Self::File>, iov: &mut IovIterDest<'_>) -> Result<usize> {
        let me = kiocb.file();

        let mut cursor = me.cursor.lock();
        let mut inner = me.shared.inner.lock();
        let msg = loop {
            if *cursor < inner.base {
                *cursor = inner.base;
            }
            let idx = *cursor - inner.base;
            if let Some(msg) = inner.messages.get(idx) {
                break msg;
            }
            if me.nonblock {
                return Err(EAGAIN);
            }
            if me.shared.condvar.wait_interruptible(&mut inner) {
                return Err(ERESTARTSYS);
            }
        };

        let written = iov.copy_to_iter(msg);
        *cursor += 1;
        Ok(written)
    }

    fn write_iter(kiocb: Kiocb<'_, Self::File>, iov: &mut IovIterSource<'_>) -> Result<usize> {
        if iov.is_empty() {
            return Ok(0);
        }

        let me = kiocb.file();
        let mut msg = KVec::new();
        let len = iov.copy_from_iter_vec(&mut msg, GFP_KERNEL)?;
        if len == 0 {
            return Ok(0);
        }

        {
            let mut inner = me.shared.inner.lock();
            inner.messages.push(msg, GFP_KERNEL)?;
        }
        me.shared.condvar.notify_all();

        Ok(len)
    }
}

#[pinned_drop]
impl PinnedDrop for RustMessageBoardFile {
    fn drop(self: Pin<&mut Self>) {
        dev_info!(self.dev, "Closing Rust Message Board\n");
    }
}
```

<details>

- **Why track `base` instead of resetting everyone to `0`?** Open file descriptors cannot safely be mutated when another process runs `> /dev/rust-message-board`. Keeping a monotonically increasing logical sequence space (`base` + index) lets already-open live readers seamlessly pick up right where they left off without bounds underflow.
- **Note on `fcntl(F_SETFL)`:** Reading `O_NONBLOCK` in `open()` handles `dd iflag=nonblock` and `open(..., O_NONBLOCK)` cleanly; a production driver can read `file.flags()` on each `read_iter()` call if it also wants to react to `fcntl(F_SETFL)` toggles after `open()`.
- **Stretch improvement (`Arc<KVec<u8>>`):** Right now `read_iter()` holds `inner.lock()` across `iov.copy_to_iter(msg)`. Storing `KVec<Arc<KVec<u8>>>` lets you clone the `Arc`, `drop(inner)`, and copy to user space without holding the shared lock.

</details>
