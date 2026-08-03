---
minutes: 25
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Driver Models & Upstreaming

The Linux kernel supports a wide variety of driver types. As Rust is introduced into existing kernel subsystems, developers can build on an expanding ecosystem of safe abstractions:

## Kinds of Drivers

1. **Functional / Class Devices (`misc`, `block`, `drm`):** Expose logical interfaces to user space.
2. **Bus Devices (`pci`, `platform`, `usb`):** Bind to hardware enumerated on physical or virtual buses.
3. **Pure Data Manipulation:** Helper modules that process protocols or algorithms without device I/O.

## Available Abstractions

The `kernel` crate provides safe wrappers across many subsystems, including:  
`kmalloc` / `kvmalloc`, `io`, `interrupt`, `workqueue`, `dma`, `configfs`, `debugfs`, `iov_iter`, and `page`.

## Going Upstream

When submitting Rust drivers and abstractions to upstream Linux:

- **Collaborate with Subsystem Maintainers:** Rust should integrate naturally into existing C subsystems rather than creating parallel silos.
- **Prefer New Drivers:** Upstream favors new hardware drivers or reference implementations over duplicate rewrites of existing C drivers.
- **Experiment and Learn:** Writing kernel code in Rust is a great opportunity to learn the language and improve kernel reliability.

<details>

- **Classroom Discussion:** Ask students which subsystems they work in and discuss how Rust abstractions can be introduced incrementally into their existing C drivers.
- Highlight that even when experimenting, kernel patches should compile cleanly across supported architectures.

</details>
