---
minutes: 20
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# The Driver Model

In Linux, a driver that interacts with user space bridges two different sides
of the kernel's Device Model:

1. **Bus Devices (`platform`, `pci`, `usb`) — Hardware Facing:**
   - Bind to hardware discovered on a bus.
   - Drivers register an ID table (e.g., Device Tree compatible strings or PCI
     IDs).
   - The kernel calls the driver's `probe` method when matching hardware is
     found, and `remove` when it is unplugged.

2. **Class Devices (`miscdevice`, `chrdev`, `input`, `drm`, `block`, `net`,
   `sound`) — User-Space Facing:**
   - Provide logical interfaces to user space, typically as device nodes in `/dev`.
     - `/dev/input/event*` for keyboards, mice, and touchscreens
     - `/dev/dri/` for graphics
     - `/dev/eth0` for networking

<details>

- Discuss how a driver is not a "bus device" or a "class device". Rather, a
  driver usually uses both!
- Usually drivers have one of each, but in principle they could have two class
  devices. Example: some GPU drivers expose both a drm device for general
  graphics, and a miscdevice for GPU-specific configuration.
- Discuss misc devices without a bus. Often these are not really drivers in the
  classical sense.

</details>
