---
minutes: 15
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Solution: Echo Miscdevice

Here is a complete implementation of the `/dev/rust_echo` character device driver:

```rust,ignore
use kernel::prelude::*;
use kernel::miscdevice::{MiscDevice, MiscDeviceOptions, MiscDeviceRegistration};
use kernel::sync::{new_mutex, Mutex};
use kernel::fs::File;

module! {
    type: EchoModule,
    name: "rust_echo",
    author: "Rust for Linux Contributors",
    description: "Echo miscdevice solution",
    license: "GPL",
}

struct EchoDevice {
    buffer: Mutex<[u8; 64]>,
}

#[vtable]
impl MiscDevice for EchoDevice {
    type Ptr = KBox<Self>;

    fn open(_file: &File, _misc: &MiscDeviceRegistration<Self>) -> Result<Self::Ptr> {
        let dev = KBox::pin_init(
            try_pin_init!(Self {
                buffer <- new_mutex!([0; 64], "EchoDevice::buffer"),
            }),
            GFP_KERNEL,
        )?;
        Ok(dev)
    }
}

struct EchoModule {
    _miscdev: Pin<KBox<MiscDeviceRegistration<EchoDevice>>>,
}

impl kernel::Module for EchoModule {
    fn init(_module: &'static ThisModule) -> Result<Self> {
        let reg = KBox::pin_init(
            MiscDeviceRegistration::register(MiscDeviceOptions {
                name: c_str!("rust_echo"),
            }),
            GFP_KERNEL,
        )?;
        pr_info!("Echo miscdevice registered\n");
        Ok(Self { _miscdev: reg })
    }
}
```
