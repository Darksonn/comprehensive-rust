---
course: Rust for Linux
session: Day 1 Morning
target_minutes: 120
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Welcome to Rust for Linux

Rust is supported as a second language for developing kernel modules and drivers in the Linux kernel. This two-day course covers how to write safe kernel modules and drivers in Rust using the `kernel` crate.

## Why Rust in Linux?

When picking a second language for the Linux kernel, it must be able to match C in efficiency while addressing historical reliability challenges:

- **Performance:** Rust is the only mature systems language that combines memory safety with C-level performance.
- **Reliability:** Rust's ownership and lifetime model prevents Use-After-Free (UAF) and concurrency bugs at compile time. Unsafe code is carefully encapsulated inside safe abstractions (for example, reference-counted pointers like `Arc` and `ARef`).
- **Productivity:** High confidence during refactoring—the *"if it compiles, it works"* philosophy reduces debugging time and logic errors.

## A "Spiral Approach" to Learning

We start Day 1 Morning by compiling, loading, and inspecting a minimal kernel module (`samples/rust/rust_minimal.rs`) to understand the basic structure of a Rust module, module parameters, and the kernel build system.

Over the four sessions of the course, we systematically build up to writing full device drivers:

- **Day 1 Morning:** Why Rust in Linux, kernel error handling (`Result`, `?`), building minimal modules (`module!`, `kernel::Module`), and setting up the QEMU test environment.
- **Day 1 Afternoon:** Kernel allocations (`KBox`, `KVec`), mutexes and synchronization, pinned initialization (`pin-init`), and writing a shared IPC character device driver (`miscdevice`).
- **Day 2 Morning:** Generating bindings with `bindgen`, handling static inline C functions, C FFI abstractions case study, the PCI subsystem, and Memory-Mapped I/O (MMIO).
- **Day 2 Afternoon:** Interrupt handling (`irq::Handler`) and writing a QEMU EDU PCI DRM driver.


## Schedule

{{%session outline}}

<details>

- **Classroom Discussion:** Ask students about difficult C kernel bugs (like UAF or unhandled return codes) they have encountered in their own work.
- Explain why we start with a working sample driver: seeing the whole picture early motivates the deep technical dives that follow.
- Encourage students to keep a terminal open in `~/linux` to cross-reference kernel sample files and documentation.

</details>
