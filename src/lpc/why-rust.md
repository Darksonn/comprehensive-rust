---
minutes: 5
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Why Rust?

- **Performance:** Matches C.
- **Reliability:**
  - Memory safety.
  - Preventing logic bugs via custom compile-time checks.
- **Productivity:**
  - _"If it compiles, it works."_
  - Ramp-up time: writing vs. reviewing.

<details>

- **The Triangle (Performance, Reliability, Productivity):** Historically you could pick at most two (C gives performance and productivity without reliability; garbage collection gives reliability and productivity without performance; formal proof checkers give performance and reliability at a steep cost to productivity).
- **Performance:** Any second language in the kernel *must* match C. Because Rust's ownership and lifetime checks happen at compile time and compile via LLVM, there is no runtime overhead.
- **Reliability = Memory safety + Logic bugs:**
  - Everyone has heard about memory safety, but Rust's ability to encode **custom domain-specific checks** into the type system is just as important.
  - Example (LWN): Wrapping untrusted user-space data in a type that forces validation before the value can be read.
  - **Example — Reference counting (`Arc<T>`, `ARef<T>`):** In C, every pointer (`struct foo *`) looks the same regardless of ownership model, and `refcount_t` saturation/underflow checks only happen at **runtime**. In Rust, pointer types encode ownership and prevent Use-After-Free, missing increments, and extra decrements **at compile time**.
  - **What if there's a bug in `arc.rs`?** Imagine if you could guarantee that all refcounting bugs in your driver are actually bugs in `include/linux/refcount.h`—isolating `unsafe` refcounting logic in one core file eliminates refcounting bugs across every safe caller and driver.
- **Productivity & Ramp-Up:**
  - Many developers find that once Rust kernel code compiles, it works on the first try.
  - **Driver authors:** Ramping up while building a concrete project takes a few months.
  - **Maintainers:** If you receive Rust patches and don't have months to study Rust, reviewing patches and walking through patch sets live with the author is a great way to learn incrementally.

</details>
