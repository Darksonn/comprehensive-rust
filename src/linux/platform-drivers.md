---
minutes: 45
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Platform Drivers

Platform drivers bind to devices described by firmware interfaces such as Device Tree or ACPI.

- **`platform::Driver` Trait:** Defines methods for probing (`probe`) and removing (`remove`) devices when matching hardware is enumerated.
- **Match Table:** Associates device tree compatibility strings with the driver implementation.

```rust,ignore
use kernel::prelude::*;

struct MyPlatformDriver;

impl kernel::platform::Driver for MyPlatformDriver {
    fn probe(_pdev: &mut kernel::platform::Device) -> Result<Self> {
        pr_info!("Probing platform device\n");
        Ok(MyPlatformDriver)
    }
}
```

<details>

- Explain how kernel probing matches compatible strings from the device tree to the registered platform driver.
- Discuss how driver private data is managed during device lifecycle transitions.

</details>
