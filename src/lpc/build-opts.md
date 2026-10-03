---
minutes: 5
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Other Build Options

```sh
# Enable extra Clippy lints:
make LLVM=1 CLIPPY=1

# Check or apply standard Rust formatting:
make LLVM=1 rustfmtcheck
make LLVM=1 rustfmt

# Generate HTML documentation for the kernel crate:
make LLVM=1 rustdoc

# Generate rust-analyzer configuration for IDE support:
make LLVM=1 rust-analyzer
```

- Online kernel Rust docs: [rust.docs.kernel.org](https://rust.docs.kernel.org/)

<details>

- **`CLIPPY=1`:** Runs the Clippy linter during kernel compilation to catch unidiomatic patterns or potential bugs.
- **`rustfmt`:** Kernel Rust code follows automated formatting enforced by `rustfmt` (you can also run `rustfmt path/to/file.rs` directly).
- **`rustdoc`:** Builds local HTML documentation into `Documentation/output/rust/rustdoc/kernel/index.html` (also hosted at `rust.docs.kernel.org`).
- **`rust-analyzer`:** Generates `rust-project.json` for IDE go-to-definition, inline type hints, and diagnostics. Re-run after adding new files or changing `.config`.
- **Testing:** Mention that `rustdoc` examples in `rust/kernel/` are compiled and executed as KUnit tests when `CONFIG_RUST_KERNEL_DOCTESTS=y` is enabled.

</details>
