---
minutes: 30
---

<!--
Copyright 2026 Google LLC
 SPDX-License-Identifier: CC-BY-4.0
-->

# Exercise: Minimal DRM Driver

In this exercise, you will create a minimal PCI DRM driver and expose a dummy IOCTL. This will set up the driver structure that you will expand in the afternoon.

---

## Tasks

1.  **Copy Starter Code:** Start with the minimal DRM driver from [Exposing via the DRM Subsystem](pci-drm.md) as your template.
2.  **Compile and Probe:**
    *   Compile the module and load it in the VM.
    *   Verify that it probes (check `dmesg`) and exposes `/dev/dri/card0` and `/dev/dri/renderD128`.
3.  **Update to EDU Device:** Change the driver to bind to the QEMU `edu` device instead of the `pci-testdev` device.
    *   *Hint:* The QEMU `edu` device has vendor ID `0x1234` and device ID `0x11e8`.
    *   Reload the driver and verify it still probes when booting QEMU with `-device edu`.
4.  **Add a Dummy IOCTL:** Follow the steps in [Defining DRM IOCTLs](drm-ioctl.md) to add a dummy `GET_ID` IOCTL that returns the hardcoded value `0x12345678` in `arg.id`.
5.  **Test the IOCTL:** Compile the `test_ioctl.c` userspace program statically, copy it to the VM, and verify that running it prints the expected dummy ID.
