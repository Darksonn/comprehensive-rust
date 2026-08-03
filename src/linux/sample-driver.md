---
minutes: 35
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# A Working Driver

To ground the concepts we will learn over the next two days, we begin with a complete, working character device driver simplified from **`samples/rust/rust_misc_device.rs`**.

In this initial version, we take out the mutex and store static data. The driver registers `/dev/rust-misc-device`; when opened and read from user space, it returns a static greeting:

```rust,ignore
use kernel::prelude::*;
use kernel::miscdevice::{MiscDevice, MiscDeviceOptions, MiscDeviceRegistration};
use kernel::fs::{File, Kiocb};
use kernel::iov::IovIterDest;
use kernel::sync::aref::ARef;
use kernel::device::Device;

module! {
    type: RustMiscDeviceModule,
    name: "rust_misc_device",
    authors: ["Rust for Linux Contributors"],
    description: "Static Rust misc device sample",
    license: "GPL",
}

#[pin_data]
struct RustMiscDeviceModule {
    #[pin]
    _miscdev: MiscDeviceRegistration<RustMiscDevice>,
}

impl kernel::InPlaceModule for RustMiscDeviceModule {
    fn init(_module: &'static ThisModule) -> impl PinInit<Self, Error> {
        pr_info!("Initialising Static Rust Misc Device Sample\n");
        let options = MiscDeviceOptions {
            name: c"rust-misc-device",
        };
        try_pin_init!(Self {
            _miscdev <- MiscDeviceRegistration::register(options),
        })
    }
}

#[pin_data(PinnedDrop)]
struct RustMiscDevice {
    dev: ARef<Device>,
}

#[vtable]
impl MiscDevice for RustMiscDevice {
    type Ptr = Pin<KBox<Self>>;

    fn open(_file: &File, misc: &MiscDeviceRegistration<Self>) -> Result<Pin<KBox<Self>>> {
        let dev = ARef::from(misc.device());
        dev_info!(dev, "Opening Static Rust Misc Device Sample\n");

        KBox::try_pin_init(
            try_pin_init! {
                RustMiscDevice { dev }
            },
            GFP_KERNEL,
        )
    }

    fn read_iter(mut kiocb: Kiocb<'_, Self::Ptr>, iov: &mut IovIterDest<'_>) -> Result<usize> {
        let me = kiocb.file();
        dev_info!(me.dev, "Reading greeting from Static Rust Misc Device\n");

        iov.simple_read_from_buffer(
            kiocb.ki_pos_mut(),
            b"Hello from Rust for Linux!\n",
        )
    }
}
```

## What We Will Unpack

Notice several unfamiliar patterns in this code that we will explore throughout the course:

1. **`ARef<Device>` & `KBox<Self>`:** Why does kernel Rust use custom pointer types instead of standard `Box` or `&Device`? *(Day 1 Afternoon)*
2. **`try_pin_init!` & `#[pin_data]`:** Why must the module and device struct be pinned and initialized in place? *(Day 1 Afternoon)*
3. **`#[vtable]`, `Kiocb`, & C FFI:** How does `MiscDevice` wrap C `file_operations`, `iov_iter`, and kernel C helpers under the hood? *(Day 2 Morning)*
4. **Adding Mutexes & State:** How do we add mutable state across reads and writes (like the full version in `samples/rust/rust_misc_device.rs`) using kernel mutexes and `new_mutex!`? *(Day 2 Morning)*
5. **Class vs. Bus Devices:** Why is this a `miscdevice` rather than a platform or PCI driver? *(Day 2 Afternoon)*

<details>

- Walk students through this static miscdevice driver. Because we omit the mutex in this initial version, the code is much shorter and easier to digest on Day 1 Morning.
- Emphasize the "no magic" rule: reassure students that every unfamiliar syntax element in this example will be explained in detail as we progress through the schedule.
- Show students that running `cat /dev/rust-misc-device` in user space invokes `open` and `read_iter`, returning `"Hello from Rust for Linux!\n"`.

</details>
