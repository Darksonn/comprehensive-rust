---
minutes: 15
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Driver Lifecycle

All devices go through the device lifecycle.

1. **Device is probed.**
2. **Device is bound.**
3. **Device unplug: device is still bound.**
    - The `unbind()` function is called. The Rust destructor runs.
4. **Device unplug: device is no longer bound.**
    - Device resources created with `devm_*` calls are freed.
5. **Device is fully unbound.**
    - Userspace might still have open fds to the device.
6. **Device is freed.**
