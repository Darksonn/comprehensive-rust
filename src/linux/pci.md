---
minutes: 15
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# The PCI Subsystem

To write a PCI driver in Rust, we implement the **`pci::Driver`** trait and declare the hardware devices our module supports.

```rust,ignore
use kernel::{device::Core, pci, prelude::*};

struct TestPciDriver;

struct TestPciData {
}

// 1. Declare the PCI Device ID table:
kernel::pci_device_table!(
    PCI_TABLE,
    <TestPciDriver as pci::Driver>::IdInfo,
    [(pci::DeviceId::from_id(pci::Vendor::REDHAT, 0x5), ())]
);

// 2. Implement the pci::Driver trait:
impl pci::Driver for TestPciDriver {
    type IdInfo = ();
    type Data<'bound> = TestPciData;

    const ID_TABLE: pci::IdTable<Self::IdInfo> = &PCI_TABLE;

    fn probe<'bound>(
        pdev: &'bound pci::Device<Core<'_>>,
        _info: Option<&'bound Self::IdInfo>,
    ) -> impl PinInit<Self::Data<'bound>, Error> + 'bound {
        dev_info!(pdev, "Probing PCI testdev device!\n");
        Ok(TestPciData {})
    }
}

// 3. Register the module:
kernel::module_pci_driver! {
    type: TestPciDriver,
    name: "rust_driver_pci",
    authors: ["Your Name"],
    description: "QEMU PCI testdev driver",
    license: "GPL v2",
}
```

## How Probing Works

- **Match:** When a PCI device with vendor `0x1b36` (Red Hat) and device ID `0x0005` (the `pci-testdev` device) is detected, the kernel calls `probe()`.
- **`pdev`:** Represents the bound PCI device, providing access to PCI config space, BARs, and IRQ allocation.
- **Resource Management:** `probe()` returns the driver's private state (`TestPciData`). When the device is unbound (e.g. on driver unload), this struct is dropped.

> **Note:** The QEMU `edu` device (used in the exercise) has vendor ID `0x1234` (QEMU) and device ID `0x11e8`.

---

## Running in QEMU

To test your drivers, you must boot the VM with the corresponding virtual PCI device virtualized by QEMU:

*   **For `pci-testdev`:** Append `-device pci-testdev` to the QEMU command line.
*   **For `edu`:** Append `-device edu` to the QEMU command line.

<details>

- Mention that `module_pci_driver!` expands to the C-side boilerplate `module_init` and `module_exit` hooks registering with the PCI core.
- Highlight that `probe()` returns a `PinInit` to initialize the driver data directly in-place, which is useful for pinning embedded locks or miscdevice registrations.

</details>
