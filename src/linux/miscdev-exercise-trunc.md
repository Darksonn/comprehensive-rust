---
minutes: 7
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Part 5: Clearing the Board (`O_TRUNC`)

In a shell, `>>` opens a file without `O_TRUNC` (append), whereas `>` passes `flags::O_TRUNC` to `open()`. Make `>` clear old messages without breaking already-open live readers!

## Task

Opening the device with `O_TRUNC` (shell `>`) should discard all existing messages, so that new readers only see messages posted afterwards. Files that were already open, including live readers sleeping in `read()`, must keep working and continue to receive new messages.

**Help:** Check the documentation of [`flags::O_TRUNC`](https://rust.docs.kernel.org/kernel/fs/file/flags/constant.O_TRUNC.html). Think about what happens to an existing file's position when the messages before it are removed.

## Testing in QEMU

```bash
rmmod rust_message_board 2>/dev/null; insmod /mnt/linux/samples/rust/rust_message_board.ko
cat /dev/rust-message-board &
echo "old 1" >> /dev/rust-message-board
echo "fresh start" > /dev/rust-message-board
dd if=/dev/rust-message-board iflag=nonblock 2>/dev/null
# Prints ONLY "fresh start", while the background `cat` receives both!
kill -INT %1
```

*Finished! Compare your implementation with [Solution: Part 5](miscdev-solution-trunc.md).*
