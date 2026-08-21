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
};

struct TestPciDriver;

#[pin_data]
struct TestPciData {
    pdev: ARef<pci::Device>,
    #[pin]
    _miscdev: MiscDeviceRegistration<TestPciMiscDevice>,
}

struct TestPciMiscDevice {}

#[vtable]
impl MiscDevice for TestPciMiscDevice {
    type Data = ();
    type Ptr = Pin<KBox<Self>>;

    fn open(_file: &File, _misc: &MiscDeviceRegistration<Self>) -> Result<Pin<KBox<Self>>> {
        KBox::try_pin_init(try_pin_init!(TestPciMiscDevice {}), GFP_KERNEL)
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
                parent: Some(pdev.as_ref()),
            };

            let miscdev_init = MiscDeviceRegistration::register(options, ());

            Ok(try_pin_init!(TestPciData {
                pdev: pdev.into(),
                _miscdev <- miscdev_init,
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
- Point out how the registration is tied to the lifetime of `TestPciData`.

</details>
