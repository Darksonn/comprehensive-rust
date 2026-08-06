---
minutes: 10
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Other build options

There are a few different useful options to build with:

## Enable extra lints

Rust has a linter called clippy that enables extra lints. Build the kernel with the linter enabled:
```sh
make LLVM=1 CLIPPY=1
```

## Code formatting

Rust code is automatically formatted using a tool called `rustfmt`.
```sh
make LLVM=1 rustfmtcheck
```
To automatically update the files on disk:
```sh
make LLVM=1 rustfmt
```
You can also invoke `rustfmt drivers/alice/rust_alice.rs` directly.

## Documentation

Documentation can be built from the Rust source files.
```sh
make LLVM=1 rustdoc
```
and open `Documentation/output/rust/rustdoc/kernel/index.html`

Can be viewed online at [rust.docs.kernel.org](https://rust.docs.kernel.org/)

## IDE support

A configuration file for Rust support in IDEs can be generated using
```
make LLVM=1 rust-analyzer
```
This should be regenerated after changing the configuration file or adding new
Kconfig options.

## Testing

Rust tests fall into two categories:

* Documentation examples
* Stand-alone kunit tests

To run the documentation examples as tests, enable
`CONFIG_RUST_KERNEL_DOCTESTS` and boot with QEMU. Standalone tests will be
covered later.
