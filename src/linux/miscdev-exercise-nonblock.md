---
minutes: 10
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Part 4: Non-Blocking Reads (`O_NONBLOCK`)

Now that `read_iter()` sleeps until a message is available, callers opening `/dev/rust-message-board` with `O_NONBLOCK` expect `EAGAIN` instead of blocking when no unread messages exist.

## Tasks

1. **Store the Flag Per Open File:** Add `nonblock: bool` to `RustMessageBoardFile`.
2. **Inspect Open Flags in `open()`:** Use `file.flags()` and `flags::O_NONBLOCK`:
   ```rust,ignore
   let nonblock = (file.flags() & flags::O_NONBLOCK) != 0;
   ```
   Pass `nonblock` into `RustMessageBoardFile` (and rename `_file` to `file`).
3. **Return `EAGAIN` in `read_iter()`:** Inside the `while *cursor >= inner.messages.len()` loop, check `if me.nonblock { return Err(EAGAIN); }` before calling `wait_interruptible`.

## Testing in QEMU

```bash
rmmod rust_message_board && insmod /mnt/linux/samples/rust/rust_message_board.ko
echo "queued message" >> /dev/rust-message-board

# Drain existing messages without blocking:
dd if=/dev/rust-message-board iflag=nonblock
# Prints "queued message", then exits with "Resource temporarily unavailable" (EAGAIN)!
```

*Want to check your work before moving on to Part 5? See [Solution: Part 4](miscdev-solution-nonblock.md).*
