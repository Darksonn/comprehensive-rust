---
session: Day 2 Afternoon
target_minutes: 180
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Day 2 Afternoon

In the afternoon session of Day 2, we dive into hardware events and write a full PCI device driver. We will cover:
- Memory-Mapped I/O (MMIO).
- IO Polling (sleeping poll).
- Interrupt Handling (MSI vectors).
- A hands-on exercise to write a PCI driver for the QEMU `edu` device.


## Schedule

{{%session outline}}

<details>

- Build incrementally on the minimal DRM PCI driver from Day 2 Morning by adding real MMIO register access (`register!`), sleeping I/O polling (`read_poll_timeout`), and MSI interrupt handling (`irq::Handler`).
- Leave time at the end to wrap up and discuss upstream Rust for Linux status and contributing.

</details>
