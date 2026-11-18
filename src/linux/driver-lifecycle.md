---
minutes: 15
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Driver Lifecycle

1. **Device is probed** (`&Device<Core>`).
2. **Device is bound** (`&Device<Bound>`).
3. **Device unplug: `unbind()` runs** (driver data destructor runs).
4. **Device unplug: `devm_*` / `Devres<T>` resources are freed.**
5. **Device is fully unbound** (user space may still hold open fds!).
6. **Device is freed** (once last `ARef<Device>` drops).

<details>

- Walk through the fundamental lifecycle mismatch in kernel drivers:
  - Hardware can be hot-unplugged (or unbound via sysfs) at any time (**Steps 3–4**), tearing down MMIO mappings and IRQs.
  - Meanwhile, user space might still have an open file descriptor (`/dev/dri/renderD128`) and call `ioctl()` (**Step 5**)!
- Explain how Rust's type system prevents use-after-unbind bugs:
  - **`ARef<Device>`:** Keeps the `struct device` allocation alive until Step 6, but does *not* allow accessing resources that require the device to be bound.
  - **`&Device<Bound>` / `'bound` lifetimes / `Devres<T>`:** Hardware resources (like `pci::Bar` or `irq::Registration`) are tied to the bound lifetime or wrapped in `Devres` / DRM's SRCU-protected registration data so they cannot be accessed after Step 4.

</details>
