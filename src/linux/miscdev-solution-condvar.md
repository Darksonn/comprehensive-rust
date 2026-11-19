---
minutes: 5
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Solution: Live Streaming with `CondVar`

```rust,ignore
// SPDX-License-Identifier: GPL-2.0
#![allow(unused_variables, unused_imports, dead_code)]

//! Rust message board character device (Milestone 3: CondVar live streaming).

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
                cursor <- new_mutex!(0),
            }),
            GFP_KERNEL,
        )
    }

    fn read_iter(kiocb: Kiocb<'_, Self::File>, iov: &mut IovIterDest<'_>) -> Result<usize> {
        let me = kiocb.file();

        let mut cursor = me.cursor.lock();
        let mut inner = me.shared.inner.lock();
        while *cursor >= inner.messages.len() {
            if me.shared.condvar.wait_interruptible(&mut inner) {
                return Err(ERESTARTSYS);
            }
        }

        let written = iov.copy_to_iter(&inner.messages[*cursor]);
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

- **Why `notify_all()`?** Every open reader maintains its own cursor and should observe every newly posted message, not just a single consumer.
- **Why drop `inner` before `notify_all()`?** Woken readers immediately try to lock `me.shared.inner`; unlocking before notifying avoids waking a reader only for it to bounce back to sleep on a contended mutex.

</details>
