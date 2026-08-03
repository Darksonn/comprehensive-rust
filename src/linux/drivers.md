---
minutes: 45
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Device Drivers

Device drivers in Rust for Linux model kernel driver registration and lifecycles using Rust traits and ownership semantics.

- **Registration Guard:** Registering a driver with a kernel subsystem returns a registration guard. Dropping the guard automatically unregisters the driver.
- **Driver State:** Associated data structure instances are tied to the lifetime of the underlying kernel object.

```rust,ignore
use kernel::prelude::*;

struct MyDriver;

impl MyDriver {
    fn register() -> Result<()> {
        // Subsystem driver registration logic.
        Ok(())
    }
}
```

<details>

- Highlight how RAII (Resource Acquisition Is Initialization) prevents resource leaks when unregistering drivers.
- Discuss how driver instances relate to kernel device structs.

</details>
