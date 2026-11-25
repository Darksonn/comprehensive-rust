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
- **Reliability:** Rust's ownership and lifetime model prevents Use-After-Free (UAF) and data races at compile time. Unsafe code is carefully encapsulated inside safe abstractions (for example, reference-counted pointers like `Arc` and `ARef`).
- **Productivity:** High confidence during refactoring—the *"if it compiles, it works"* philosophy reduces debugging time and logic errors.

## Overview of the Course

We start Day 1 Morning by compiling, loading, and inspecting a minimal kernel module (`samples/rust/rust_minimal.rs`) to understand the basic structure of a Rust module, module parameters, and the kernel build system.

Over the four sessions of the course, we systematically build up to writing full device drivers:

- **Day 1 Morning:** Why Rust in Linux, how the kernel's Rust layers fit together, setting up the QEMU test environment, and building your own minimal kernel module.
- **Day 1 Afternoon:** The driver model, kernel allocations (`KBox`, `KVec`), mutexes and condition variables, and pinned initialization (`pin-init`).
- **Day 1 Exercise:** A character device message board (`/dev/rust-message-board`), built up in five small milestones from a shared buffer to live streaming with `CondVar`.
- **Day 2 Morning:** Generating bindings with `bindgen`, static inline C helpers, the credentials case study, and the PCI and DRM subsystems.
- **Day 2 Afternoon:** Memory-mapped I/O with the `register!` macro, I/O polling, and interrupt handling (`irq::Handler`).
- **Day 2 Exercise:** A PCI DRM driver for the QEMU `edu` device, built up from a minimal DRM driver to device ID/liveness, factorial computation, and interrupts.

## Schedule

{{%session outline}}

<details>

- **Classroom Discussion:** Ask students about difficult C kernel bugs (like UAF or unhandled return codes) they have encountered in their own work.
- Explain why we start with a working sample driver: seeing the whole picture early motivates the deep technical dives that follow.
- Encourage students to keep a terminal open in `~/learn-rust/linux` to cross-reference kernel sample files and documentation.

</details>
