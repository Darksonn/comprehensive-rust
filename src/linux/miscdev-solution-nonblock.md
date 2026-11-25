---
minutes: 5
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Solution: Part 4 — Non-Blocking Reads (`O_NONBLOCK`)

Changes compared to Part 3:

```diff
@@ -85,14 +85,16 @@
     dev: ARef<Device>,
     #[pin]
     cursor: Mutex<usize>,
+    nonblock: bool,
 }
 
 #[vtable]
 impl MiscDevice for RustMessageBoard {
     type File = Pin<KBox<RustMessageBoardFile>>;
 
-    fn open(_file: &File, misc: &MiscDeviceRegistration<Self>) -> Result<Self::File> {
+    fn open(file: &File, misc: &MiscDeviceRegistration<Self>) -> Result<Self::File> {
         let dev = ARef::from(misc.device());
+        let nonblock = (file.flags() & flags::O_NONBLOCK) != 0;
 
         dev_info!(dev, "Opening Rust Message Board\n");
 
@@ -101,6 +103,7 @@
                 shared: Arc::clone(&misc.shared),
                 dev,
                 cursor <- new_mutex!(0),
+                nonblock,
             }),
             GFP_KERNEL,
         )
@@ -112,6 +115,9 @@
         let mut cursor = me.cursor.lock();
         let mut inner = me.shared.inner.lock();
         while *cursor >= inner.messages.len() {
+            if me.nonblock {
+                return Err(EAGAIN);
+            }
             if me.shared.condvar.wait_interruptible(&mut inner) {
                 return Err(ERESTARTSYS);
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

        dev_info!(dev, "Opening Rust Message Board\n");

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
        while *cursor >= inner.messages.len() {
            if me.nonblock {
                return Err(EAGAIN);
            }
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

*Continue to [Part 5: Clearing the Board (`O_TRUNC`)](miscdev-exercise-trunc.md).*
