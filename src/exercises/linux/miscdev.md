---
minutes: 45
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Exercise: Echo Miscdevice

In this exercise, we will implement an echo character device driver (`/dev/rust_echo`) using `kernel::miscdevice::MiscDevice`.

The driver maintains an internal message buffer protected by a pinned `Mutex`. When user space writes to `/dev/rust_echo`, the driver stores the bytes; when user space reads, the driver returns the stored bytes.

## Starter Code

```rust,ignore
use kernel::prelude::*;
use kernel::miscdevice::{MiscDevice, MiscDeviceOptions, MiscDeviceRegistration};
use kernel::sync::{new_mutex, Mutex};
use kernel::fs::File;

module! {
    type: EchoModule,
    name: "rust_echo",
    author: "Rust for Linux Contributors",
    description: "Echo miscdevice exercise",
    license: "GPL",
}

struct EchoDevice {
    buffer: Mutex<[u8; 64]>,
}

#[vtable]
impl MiscDevice for EchoDevice {
    type Ptr = KBox<Self>;

    fn open(_file: &File, _misc: &MiscDeviceRegistration<Self>) -> Result<Self::Ptr> {
        // TODO: Allocate a new EchoDevice using KBox::pin_init and return it.
        todo!()
    }
}

struct EchoModule {
    _miscdev: Pin<KBox<MiscDeviceRegistration<EchoDevice>>>,
}

impl kernel::Module for EchoModule {
    fn init(_module: &'static ThisModule) -> Result<Self> {
        // TODO: Register the miscdevice with name "rust_echo".
        todo!()
    }
}
```

<details>

- Encourage students to inspect `/usr/local/google/home/aliceryhl/linux/rust/kernel/miscdevice.rs` to see the full `MiscDevice` trait definition.
- Remind students that `KBox::pin_init` is required when allocating structs containing a `Mutex`.

</details>
