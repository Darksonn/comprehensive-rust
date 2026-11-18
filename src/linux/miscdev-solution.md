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

use core::fmt::Write;
use kernel::{
    device::Device,
    fs::{File, Kiocb},
    iov::{IovIterDest, IovIterSource},
    miscdevice::{MiscDevice, MiscDeviceOptions, MiscDeviceRegistration},
    new_mutex,
    prelude::*,
    sync::{aref::ARef, Arc, Mutex},
};

module! {
    type: RustMiscDeviceModule,
    name: "rust_misc_device",
    authors: ["Rust for Linux Course"],
    description: "Shared IPC Message Board Sample",
    license: "GPL",
}

#[pin_data]
struct RustMiscDeviceModule {
    #[pin]
    _miscdev: MiscDeviceRegistration<RustMiscDevice>,
}

impl kernel::InPlaceModule for RustMiscDeviceModule {
    fn init(_module: &'static ThisModule) -> impl PinInit<Self, Error> {
        pin_init::pin_init_scope(move || {
            pr_info!("Initializing Shared Rust Misc Device Sample\n");

            let shared = Arc::pin_init(
                try_pin_init!(SharedState {
                    inner <- new_mutex!(Inner {
                        buffer: KVVec::new(),
                    }),
                }),
                GFP_KERNEL,
            )?;

            let options = MiscDeviceOptions {
                name: c"rust-misc-device",
            };

            Ok(try_pin_init!(Self {
                // SAFETY: `_miscdev` is stored in `RustMiscDeviceModule` and dropped when the
                // module is unloaded; `mem::forget()` is never called on it.
                _miscdev <- unsafe {
                    MiscDeviceRegistration::register(
                        options,
                        try_pin_init!(RustMiscDevice { shared }),
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
struct RustMiscDevice {
    shared: Arc<SharedState>,
}

/// Data that is stored per file for this miscdevice.
#[pin_data]
struct RustMiscDeviceFile {
    shared: Arc<SharedState>,
    dev: ARef<Device>,
}

#[vtable]
impl MiscDevice for RustMiscDevice {
    type File = Pin<KBox<RustMiscDeviceFile>>;

    fn open(_file: &File, misc: &MiscDeviceRegistration<Self>) -> Result<Self::File> {
        let dev = ARef::from(misc.device());

        KBox::try_pin_init(
            try_pin_init!(RustMiscDeviceFile {
                shared: Arc::clone(&misc.shared),
                dev,
            }),
            GFP_KERNEL,
        )
    }

    fn write_iter(mut kiocb: Kiocb<'_, Self::File>, iov: &mut IovIterSource<'_>) -> Result<usize> {
        let me = kiocb.file();
        let mut data = KVec::new();
        iov.copy_from_iter_vec(&mut data, GFP_KERNEL)?;

        let pid = current!().pid();

        // Format prefix onto stack
        let mut buf = [0u8; 32];
        let mut formatter = kernel::str::Formatter::new(&mut buf);
        let _ = write!(formatter, "[PID {}]: ", pid);
        let prefix = &buf[..formatter.bytes_written()];

        let mut guard = me.shared.inner.lock();
        guard.buffer.clear();
        guard.buffer.extend_from_slice(prefix, GFP_KERNEL)?;
        guard.buffer.extend_from_slice(&data, GFP_KERNEL)?;

        *kiocb.ki_pos_mut() = 0;

        Ok(data.len())
    }

    fn read_iter(mut kiocb: Kiocb<'_, Self::File>, iov: &mut IovIterDest<'_>) -> Result<usize> {
        let me = kiocb.file();
        let inner = me.shared.inner.lock();
        let read = iov.simple_read_from_buffer(kiocb.ki_pos_mut(), &inner.buffer)?;
        Ok(read)
    }
}
```

<details>

- **Architectural Pattern:** Separating registration-wide state (`RustMiscDevice` holding `Arc<SharedState>`) from per-open file state (`RustMiscDeviceFile`) is the foundation used by character device and IPC drivers.
- Note that if no per-file fields are needed beyond `Arc<SharedState>`, you can even set `type File = Arc<SharedState>;` and return `Ok(Arc::clone(&misc.shared))` directly from `open()`!

</details>
