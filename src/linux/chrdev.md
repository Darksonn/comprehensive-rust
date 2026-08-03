---
minutes: 45
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Character Devices

Character devices expose kernel functionality to user space via `/dev` nodes. Rust for Linux abstracts file operations through traits:

- **`FileOpener`:** Handles opening device nodes and initializing per-file instance data.
- **`FileOperations`:** Defines callbacks for file operations such as `read`, `write`, and `ioctl`.
- **`Registration`:** Manages registering character device major and minor numbers with the kernel.

```rust,ignore
use kernel::prelude::*;

struct MyFile;

#[vtable]
impl kernel::file::Operations for MyFile {
    // Override read, write, ioctl methods as needed.
}
```

<details>

- Explain the role of `#[vtable]` in generating C-compatible function pointers for the kernel's `file_operations` struct.
- Emphasize how Rust ensures safe user-space memory access during read and write operations.

</details>
