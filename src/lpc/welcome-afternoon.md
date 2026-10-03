---
session: Afternoon
target_minutes: 210
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Afternoon Session: Hands-On PCI & DRM Driver

Writing a PCI/DRM driver for the QEMU `edu` device.

## Schedule

{{%session outline}}

<details>

- In the afternoon session, we apply the abstractions from the morning to build a functional PCI/DRM device driver for the QEMU `edu` hardware device ([`samples/rust/rust_driver_pci_edu_drm.rs`](https://github.com/Darksonn/linux/blob/rfl-course-edu/samples/rust/rust_driver_pci_edu_drm.rs)).
- We will cover the driver model, PCI probing, DRM registration and IOCTLs, Memory-Mapped I/O (`register!`), sleeping I/O polling (`read_poll_timeout`), and MSI interrupts (`irq::Handler`).

</details>
