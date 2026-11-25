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

## Task

Change `write_iter()` so that each write is appended to the shared buffer instead of replacing its contents.

**Help:** Check the documentation of [`IovIterSource::copy_from_iter_vec`](https://rust.docs.kernel.org/kernel/iov/struct.IovIterSource.html#method.copy_from_iter_vec).

## Testing in QEMU

Build the module and test that multiple writes accumulate:

```bash
# On the host:
make LLVM=1 samples/rust/rust_message_board.ko

# Inside QEMU:
rmmod rust_message_board 2>/dev/null; insmod /mnt/linux/samples/rust/rust_message_board.ko
echo "hello" >> /dev/rust-message-board
echo "world" >> /dev/rust-message-board
cat /dev/rust-message-board
# Output:
# hello
# world
```

*Want to check your work before moving on to Part 2? See [Solution: Part 1](miscdev-solution-append.md).*
