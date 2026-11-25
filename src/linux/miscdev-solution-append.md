---
minutes: 5
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Solution: Part 1 — Append Instead of Overwrite

Removing `inner.buffer.clear()` makes `iov.copy_from_iter_vec()` append each incoming write onto `inner.buffer`.

Changes compared to the starter stub:

```diff
@@ -1,7 +1,7 @@
 // SPDX-License-Identifier: GPL-2.0
 #![allow(unused_variables, unused_imports, dead_code)]
 
-//! Rust message board character device exercise starter stub.
+//! Rust message board character device.
 
 use kernel::{
     device::Device,
@@ -109,13 +109,11 @@
         Ok(read)
     }
 
-    fn write_iter(mut kiocb: Kiocb<'_, Self::File>, iov: &mut IovIterSource<'_>) -> Result<usize> {
+    fn write_iter(kiocb: Kiocb<'_, Self::File>, iov: &mut IovIterSource<'_>) -> Result<usize> {
         let me = kiocb.file();
 
         let mut inner = me.shared.inner.lock();
-        inner.buffer.clear();
         let len = iov.copy_from_iter_vec(&mut inner.buffer, GFP_KERNEL)?;
-        *kiocb.ki_pos_mut() = 0;
 
         Ok(len)
     }
```

Complete solution:

```rust,ignore
// SPDX-License-Identifier: GPL-2.0
#![allow(unused_variables, unused_imports, dead_code)]

//! Rust message board character device.

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
            pr_info!("Initialising Rust Message Board\n");

            let shared = Arc::pin_init(
                try_pin_init!(SharedState {
                    inner <- new_mutex!(Inner {
                        buffer: KVVec::new(),
                    }),
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
    buffer: KVVec<u8>,
}

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

        dev_info!(dev, "Opening Rust Message Board\n");

        KBox::try_pin_init(
            try_pin_init!(RustMessageBoardFile {
                shared: Arc::clone(&misc.shared),
                dev,
            }),
            GFP_KERNEL,
        )
    }

    fn read_iter(mut kiocb: Kiocb<'_, Self::File>, iov: &mut IovIterDest<'_>) -> Result<usize> {
        let me = kiocb.file();

        let inner = me.shared.inner.lock();
        let read = iov.simple_read_from_buffer(kiocb.ki_pos_mut(), &inner.buffer)?;

        Ok(read)
    }

    fn write_iter(kiocb: Kiocb<'_, Self::File>, iov: &mut IovIterSource<'_>) -> Result<usize> {
        let me = kiocb.file();

        let mut inner = me.shared.inner.lock();
        let len = iov.copy_from_iter_vec(&mut inner.buffer, GFP_KERNEL)?;

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

*Continue to [Part 2: Discrete Messages & Per-File Cursor](miscdev-exercise-cursor.md).*
