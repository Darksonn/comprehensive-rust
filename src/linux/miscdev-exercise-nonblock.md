---
minutes: 10
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Part 4: Non-Blocking Reads (`O_NONBLOCK`)

Now that `read_iter()` sleeps until a message is available, callers opening `/dev/rust-message-board` with `O_NONBLOCK` expect `EAGAIN` instead of blocking when no unread messages exist.

## Task

When the device is opened with `O_NONBLOCK`, a `read()` that would otherwise sleep (because that file has already read every message) should fail immediately with `EAGAIN`. Blocking files should keep the behavior from Part 3.

**Help:** Check the documentation of [`File::flags`](https://rust.docs.kernel.org/kernel/fs/file/struct.LocalFile.html#method.flags) and [`flags::O_NONBLOCK`](https://rust.docs.kernel.org/kernel/fs/file/flags/constant.O_NONBLOCK.html).

## Testing in QEMU

```bash
rmmod rust_message_board 2>/dev/null; insmod /mnt/linux/samples/rust/rust_message_board.ko
echo "queued message" >> /dev/rust-message-board

# Drain existing messages without blocking:
dd if=/dev/rust-message-board iflag=nonblock
# Prints "queued message", then exits with "Resource temporarily unavailable" (EAGAIN)!
```

*Want to check your work before moving on to Part 5? See [Solution: Part 4](miscdev-solution-nonblock.md).*
