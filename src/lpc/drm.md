---
minutes: 5
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# The DRM Subsystem

- **Direct Rendering Manager (DRM):** Subsystem for GPUs and compute/AI accelerators.
- **Device nodes:**
  - `/dev/dri/cardX` (Primary/display node)
  - `/dev/dri/renderDXX` (Render/compute node, unprivileged)
- **SRCU lifetime protection:** Safely bridges PCI `unbind` with open user-space file descriptors.

<details>

- Modern accelerators (including AI/ML accelerators) expose themselves as DRM devices rather than raw character devices.
- Unlike `miscdevice` (where you pick a static node name), DRM dynamically assigns minor numbers (`/dev/dri/card0`, `/dev/dri/renderD128`).
- **Why DRM for our PCI driver?** The Rust DRM abstraction uses sleepable Read-Copy-Update (SRCU) to guard `RegistrationData<'drm>`. When an `ioctl` runs, DRM holds an SRCU read lock ensuring that resources borrowed from the PCI `'bound` scope (like MMIO BARs and IRQs) remain valid for the duration of the `ioctl`, and blocks `unbind` until in-flight `ioctl` calls finish.

</details>
