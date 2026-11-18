---
minutes: 15
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# The PCI Subsystem

```rust,ignore
use kernel::{device::Core, pci, prelude::*};

struct TestPciDriver;

#[pin_data]
struct TestPciData {}

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
        dev_info!(pdev, "Probing PCI testdev device!\n");
        try_pin_init!(TestPciData {})
    }
}

kernel::module_pci_driver! {
    type: TestPciDriver,
    name: "rust_driver_pci",
    authors: ["Your Name"],
    description: "QEMU PCI testdev driver",
    license: "GPL v2",
}
```

<details>

- **Matching hardware (`pci_device_table!`):** When a PCI device matching Vendor `0x1b36` (Red Hat) and Device ID `0x0005` (`pci-testdev`) is discovered, the kernel calls `probe()`. (For the QEMU `edu` device in our exercise, the ID is `pci::Vendor::QEMU`, `0x11e8`.)
- **`probe()` returns `impl PinInit<Self::Data<'bound>, Error>`:** Initializes the driver's private state in-place, parameterized by `'bound` (the lifetime for which the driver is bound to the PCI device). When the device is unbound, `TestPciData` is dropped.
- **`module_pci_driver!`:** Generates the `module_init` / `module_exit` boilerplate registering `TestPciDriver` with the PCI core.
- **QEMU flags:** `-device pci-testdev` or `-device edu`.

</details>
