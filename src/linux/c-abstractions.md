---
minutes: 10
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Kernel Abstractions

<p align="center">
  <img src="safeunsafe.svg" alt="Safe Rust Driver, Kernel Abstraction Layer, and C Subsystems" />
</p>

<details>

- Day 1 focused on *using* safe abstractions in the `kernel` crate (`KBox`, `Mutex`, `CondVar`, `MiscDevice`).
- This section zooms in on the middle layers of the diagram: how `bindgen` generates raw FFI declarations, how `rust/helpers/` bridges `static inline` C functions, and how `rust/kernel/` wraps them in safe types (`Credential`, `File::cred`).

</details>
