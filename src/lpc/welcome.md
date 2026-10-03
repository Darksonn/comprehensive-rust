---
course: Rust for Linux (LPC)
session: Morning
target_minutes: 180
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Welcome to Rust for Linux (LPC)

Writing and reviewing safe kernel abstractions and drivers in Rust.

## Schedule

{{%session outline}}

<details>

- Welcome attendees to the single-day Rust for Linux workshop at LPC.
- Confirm everyone has their pre-course setup ready (`~/learn-rust/linux` built on `rfl-course-edu` and `debian.img` downloaded).
- Briefly preview the arc of the day:
  - **Morning:** Why Rust in the kernel, how API design enforces correctness (`Mutex`, fallible allocations), and how we wrap C subsystems in `unsafe` Rust (`bindgen`, C helpers, `Credential`).
  - **Afternoon:** The driver model and hands-on writing a PCI/DRM driver for the QEMU `edu` device (MMIO registers, sleeping poll, MSI interrupts).

</details>
