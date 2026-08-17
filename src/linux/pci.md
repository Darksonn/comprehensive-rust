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
use kernel::pci;

struct EduDriverData;

struct EduDriver;

// 1. Declare the PCI Device ID table:
kernel::pci_device_table!(
    PCI_TABLE,
    <EduDriver as pci::Driver>::IdInfo,
    [(pci::DeviceId::from_id(pci::Vendor::QEMU, 0x11e8), ())]
);

// 2. Implement the pci::Driver trait:
impl pci::Driver for EduDriver {
    type IdInfo = ();
    type Data<'bound> = EduDriverData;

    const ID_TABLE: pci::IdTable<Self::IdInfo> = &PCI_TABLE;

    fn probe<'bound>(
        pdev: &'bound pci::Device<Core<'_>>,
        _info: Option<&'bound Self::IdInfo>,
    ) -> impl PinInit<Self::Data<'bound>, Error> + 'bound {
        dev_info!(pdev, "Probing QEMU EDU PCI device!\n");
        Ok(EduDriverData)
    }
}

// 3. Register the module:
kernel::module_pci_driver! {
    type: EduDriver,
    name: "rust_driver_pci",
    authors: ["Alice Ryhl"],
    description: "QEMU PCI EDU driver",
    license: "GPL v2",
}
```

## How Probing Works

- **Match:** When a PCI device with vendor `0x1234` (QEMU) and device ID `0x11e8` is detected, the kernel calls `probe()`.
- **`pdev`:** Represents the bound PCI device, providing access to PCI config space, BARs, and IRQ allocation.
- **Resource Management:** `probe()` returns the driver's private state (`EduDriverData`). When the device is unbound (e.g. on driver unload), this struct is dropped.

<details>

- Mention that `module_pci_driver!` expands to the C-side boilerplate `module_init` and `module_exit` hooks registering with the PCI core.
- Highlight that `probe()` returns a `PinInit` to initialize the driver data directly in-place, which is useful for pinning embedded locks or miscdevice registrations.

</details>
