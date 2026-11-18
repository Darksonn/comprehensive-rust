---
minutes: 10
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Exposing via miscdevice

A simple character device interface (`/dev/pci-testdev`) can be added by registering a `MiscDeviceRegistration` within our PCI driver state.

```rust,ignore
// SPDX-License-Identifier: GPL-2.0
//! PCI driver exposing a miscdevice.

use kernel::{
    device::Core,
    fs::File,
    miscdevice::{MiscDevice, MiscDeviceOptions, MiscDeviceRegistration},
    pci,
    prelude::*,
    sync::aref::ARef,
};

struct TestPciDriver;

#[pin_data]
struct TestPciData {
    pdev: ARef<pci::Device>,
    #[pin]
    _miscdev: MiscDeviceRegistration<TestPciMiscDevice>,
}

#[pin_data]
struct TestPciMiscDevice {}

#[pin_data]
struct TestPciMiscFile {}

#[vtable]
impl MiscDevice for TestPciMiscDevice {
    type File = Pin<KBox<TestPciMiscFile>>;

    fn open(_file: &File, _misc: &MiscDeviceRegistration<Self>) -> Result<Self::File> {
        KBox::try_pin_init(try_pin_init!(TestPciMiscFile {}), GFP_KERNEL)
    }
}

kernel::pci_device_table!(
    PCI_TABLE,
    <TestPciDriver as pci::Driver>::IdInfo,
    [(pci::DeviceId::from_id(pci::Vendor::REDHAT, 0x5), ())]
);

impl pci::Driver for TestPciDriver {
    type IdInfo = ();
    type Data<'bound> = TestPciData;

    const ID_TABLE: pci::IdTable<Self::IdInfo> = &PCI_TABLE;

    fn probe<'bound>(
        pdev: &'bound pci::Device<Core<'_>>,
        _info: Option<&'bound Self::IdInfo>,
    ) -> impl PinInit<Self::Data<'bound>, Error> + 'bound {
        pin_init::pin_init_scope(move || {
            dev_info!(pdev, "Probing PCI testdev misc device!\n");

            pdev.enable_device_mem()?;
            pdev.set_master();

            let options = MiscDeviceOptions {
                name: c"pci-testdev",
            };

            Ok(try_pin_init!(TestPciData {
                pdev: pdev.into(),
                // SAFETY: `_miscdev` is stored in `TestPciData` and dropped when the PCI
                // device is unbound; `mem::forget()` is never called on it.
                _miscdev <- unsafe {
                    MiscDeviceRegistration::register(options, try_pin_init!(TestPciMiscDevice {}))
                },
            }))
        })
    }
}

kernel::module_pci_driver! {
    type: TestPciDriver,
    name: "rust_driver_pci_testdev_misc",
    authors: ["Your Name"],
    description: "PCI testdev misc driver",
    license: "GPL v2",
}
```

<details>

- Explain that `miscdevice` is a simplified interface for character devices that doesn't require allocating major numbers manually.
- Point out how `_miscdev` is pinned inside `TestPciData`, so the character device is automatically unregistered when the PCI device is unbound.
- However, notice that `MiscDevice::File` must be `'static` (because an open file descriptor can outlive `misc_deregister`), which means `TestPciMiscDevice` cannot directly borrow `'bound` PCI resources like `pci::Bar<'bound>` without wrapping them in `Devres<T>` and `Arc<T>`. Next, we'll see how the DRM subsystem solves this with `'drm` lifetimes and SRCU.

</details>
