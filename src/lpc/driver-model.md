---
minutes: 15
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# The Driver Model

1. **Bus Devices (Hardware-Facing):**
   - `pci`, `platform`, `auxiliary`, `faux`, (`usb`, `i2c`)
   - Match hardware via ID tables (`probe` / `remove`).

2. **Class Devices (User-Space-Facing):**
   - `miscdevice`, `drm`, `block`, `net::phy`, (`pwm`, `hid`)
   - Expose interfaces to user space (`/dev/dri/*`, `/dev/input/*`, etc.).

<details>

- A driver is rarely *just* a "bus device" or *just* a "class device"—most hardware drivers implement both!
- During `probe()` on the bus device (e.g. `pci::Driver::probe`), the driver initializes hardware resources (BARs, IRQs) and registers a class device (e.g. `drm::Registration` or `MiscDeviceRegistration`) so user space can interact with it.
- Also mention pure data-manipulation modules (such as the DRM panic QR code generator) that don't bind to a bus or class device at all.

</details>
