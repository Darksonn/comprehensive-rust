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

We start Day 1 Morning by compiling, loading, and inspecting a simplified, static version of a character device driver from the kernel source tree (`samples/rust/rust_misc_device.rs`), where we omit the mutex and store static data to focus on driver registration and file operations.

Over the four sessions of the course, we systematically unpack the kernel concepts used in that sample driver:

- **Day 1 Morning:** Why Rust in Linux, kernel error handling (`Result`, `?`), and building minimal modules (`module!`, `kernel::Module`).
- **Day 1 Afternoon:** Why kernel pointer types (`KBox`, `KVBox`, `Arc`, `ARef`) and pinned initialization (`pin-init`, `try_pin_init!`) are required for kernel memory.
- **Day 2 Morning:** Kernel locking rules (`Mutex`, `SpinLock`), C FFI (`bindings_helper.h` and `rust/helpers/`), and character device file operations.
- **Day 2 Afternoon:** The Linux Device Model (bus vs. class devices) and writing your own custom `miscdevice` driver from scratch.

## Schedule

{{%session outline}}

<details>

- **Classroom Discussion:** Ask students about difficult C kernel bugs (like UAF or unhandled return codes) they have encountered in their own work.
- Explain why we start with a working sample driver: seeing the whole picture early motivates the deep technical dives that follow.
- Encourage students to keep a terminal open in `~/linux` to cross-reference kernel sample files and documentation.

</details>
