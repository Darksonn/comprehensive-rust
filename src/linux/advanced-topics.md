---
minutes: 45
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Advanced Topics

As Rust for Linux evolves, new abstractions continue to be added to support diverse kernel subsystems:

- **Workqueues and Timers:** Executing deferred work safely in kernel context.
- **Interrupt Handling:** Safe abstractions over top-half and bottom-half interrupt handlers.
- **Custom FFI Bindings:** When wrapping C APIs not yet exposed by the `kernel` crate, developers can add new safe wrappers following kernel soundness rules.
- **Contributing Upstream:** Guidelines for submitting Rust abstractions to the Linux kernel mailing lists.

<details>

- Encourage students to review upstream RFCs and patches on the Rust for Linux mailing list.
- Emphasize the rule that every `unsafe` block wrapping C kernel code must document its safety preconditions.

</details>
