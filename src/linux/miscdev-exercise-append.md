---
minutes: 5
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Part 1: Append Instead of Overwrite

Inspect the shared-buffer starter stub at [`samples/rust/rust_message_board.rs`](https://github.com/Darksonn/linux/blob/rfl-course-edu/samples/rust/rust_message_board.rs) (exposed as `/dev/rust-message-board`).

Right now, every `write_iter()` clears the shared buffer and resets the file offset, so each new write overwrites the previous message.

## Tasks

1. In `write_iter()`, remove `inner.buffer.clear()` so incoming writes append to `inner.buffer`.
2. Remove `*kiocb.ki_pos_mut() = 0;` and change `mut kiocb` to `kiocb` in `write_iter()`.

## Testing in QEMU

Build the module and test that multiple writes accumulate:

```bash
# On the host:
make LLVM=1 samples/rust/rust_message_board.ko

# Inside QEMU:
insmod /mnt/linux/samples/rust/rust_message_board.ko
echo "hello" > /dev/rust-message-board
echo "world" > /dev/rust-message-board
cat /dev/rust-message-board
# Output:
# hello
# world
```

*Want to check your work before moving on to Part 2? See [Solution: Part 1](miscdev-solution-append.md).*
