---
minutes: 10
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Exposing via the DRM Subsystem

For graphics cards and accelerators, the **Direct Rendering Manager (DRM)** subsystem is preferred. A DRM device (`/dev/dri/cardX`) is registered using the `drm::Driver` trait.

```rust,ignore
// SPDX-License-Identifier: GPL-2.0
//! Minimal DRM PCI driver (no MMIO or IOCTLs).

use kernel::{
    device::{Core, DeviceContext},
    drm,
    pci,
    prelude::*,
    sync::aref::ARef,
};

struct TestPciDriver;

#[pin_data]
struct TestPciData<'bound> {
    pdev: ARef<pci::Device>,
    _reg: drm::Registration<'bound, TestPciDriver>,
}

#[pin_data]
struct TestDrmData {}

struct TestFile;

#[pin_data]
struct TestObject {}

impl pci::Driver for TestPciDriver {
    type IdInfo = ();
    type Data<'bound> = TestPciData<'bound>;

    const ID_TABLE: pci::IdTable<Self::IdInfo> = &PCI_TABLE;

    fn probe<'bound>(
        pdev: &'bound pci::Device<Core<'_>>,
        _info: Option<&'bound Self::IdInfo>,
    ) -> impl PinInit<Self::Data<'bound>, Error> + 'bound {
        pin_init::pin_init_scope(move || {
            pdev.enable_device_mem()?;
            pdev.set_master();

            let unreg_dev = drm::UnregisteredDevice::<TestPciDriver>::new(pdev, Ok(()))?;

            // Initialize the shared DRM registration data
            let reg_data = try_pin_init!(TestDrmData {});

            let reg = unsafe {
                drm::Registration::new(pdev.as_ref(), unreg_dev, reg_data, 0)?
            };

            Ok(try_pin_init!(TestPciData {
                pdev: pdev.into(),
                _reg: reg,
            }))
        })
    }
}

#[vtable]
impl drm::Driver for TestPciDriver {
    type Data = ();
    type RegistrationData<'drm> = TestDrmData;
    type File = TestFile;
    type Object = drm::gem::Object<TestObject>;
    type ParentDevice<Ctx: DeviceContext> = pci::Device<Ctx>;

    const INFO: drm::DriverInfo = drm::DriverInfo {
        major: 1,
        minor: 0,
        patchlevel: 0,
        name: c"pci-testdev-drm",
        desc: c"PCI Testdev DRM Driver",
    };

    const FEAT_RENDER: bool = true;

    kernel::declare_drm_ioctls! {}
}

impl drm::file::DriverFile for TestFile {
    type Driver = TestPciDriver;

    fn open(_dev: &drm::Device<TestPciDriver>) -> Result<Pin<KBox<Self>>> {
        Ok(KBox::new(Self, GFP_KERNEL)?.into())
    }
}

impl drm::gem::DriverObject for TestObject {
    type Driver = TestPciDriver;
    type Args = ();

    fn new(
        _dev: &drm::Device<TestPciDriver>,
        _size: usize,
        _args: Self::Args,
    ) -> impl PinInit<Self, Error> {
        try_pin_init!(TestObject {})
    }
}

kernel::pci_device_table!(
    PCI_TABLE,
    <TestPciDriver as pci::Driver>::IdInfo,
    [(pci::DeviceId::from_id(pci::Vendor::REDHAT, 0x5), ())]
);

kernel::module_pci_driver! {
    type: TestPciDriver,
    name: "rust_driver_pci_testdev_drm",
    authors: ["Your Name"],
    description: "PCI Testdev DRM driver",
    license: "GPL v2",
}
```

---

## Device Node Names

Unlike `miscdevice` (which requires a hardcoded string or manual unique name generation), the DRM subsystem manages device node filenames automatically:

*   **Dynamic Minor Allocation:** The DRM core assigns a sequential minor number to each registered device instance.
*   **Automatic `/dev` Nodes:**
    *   `/dev/dri/cardX` (Primary node, e.g., `/dev/dri/card0`)
    *   `/dev/dri/renderDXX` (Render node, e.g., `/dev/dri/renderD128`)

<details>

- Explain that `drm::gem::Object` handles GPU memory allocations, which are also tied into the DRM class device interface.
- Highlight the contrast: `miscdevice` is lightweight and simple, but has fewer built-in memory protection guarantees during dynamic unbind than the DRM subsystem's SRCU-guarded registry.

</details>
