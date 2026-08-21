---
minutes: 5
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# The DRM Subsystem

The **Direct Rendering Manager (DRM)** is the Linux subsystem responsible for interfacing with GPUs and other graphics/computation accelerators.

*   **Modern Interface:** Modern accelerators (including AI/ML accelerators) expose themselves as DRM devices rather than simple character devices.
*   **Split Nodes:** Exposes different device nodes for different use cases:
    *   `/dev/dri/cardX` (Master node, requires privileges, used for display configuration).
    *   `/dev/dri/renderDXX` (Render node, non-privileged, used for compute/rendering).
*   **SRCU Protection:** The Rust DRM abstraction uses sleepable Read-Copy-Update (SRCU) to ensure that device data remains valid during userspace operations, even if the hardware is hot-unplugged.
