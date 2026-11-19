---
minutes: 7
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Part 5: Clearing the Board (`O_TRUNC`)

In a shell, `>>` opens a file without `O_TRUNC` (append), whereas `>` passes `flags::O_TRUNC` to `open()`. Make `>` clear old messages without breaking already-open live readers!

## Tasks

1. **Track Cleared Messages:** Add `base: usize` (initialized to `0`) to `Inner` to record how many older messages have been discarded.
2. **Clear on `O_TRUNC` in `open()`:**
   ```rust,ignore
   if (file.flags() & flags::O_TRUNC) != 0 {
       let mut inner = misc.shared.inner.lock();
       inner.base += inner.messages.len();
       inner.messages.clear();
   }
   ```
3. **Adjust Indexing in `read_iter()`:**
   - Replace the `while` loop with a `loop` that clamps `*cursor = (*cursor).max(inner.base);`.
   - When `*cursor < inner.base + inner.messages.len()`, `break` and copy `&inner.messages[*cursor - inner.base]`.
   - Otherwise check `me.nonblock` (`Err(EAGAIN)`) and wait on `me.shared.condvar`.

## Testing in QEMU

```bash
rmmod rust_message_board && insmod /mnt/linux/samples/rust/rust_message_board.ko
cat /dev/rust-message-board &
echo "old 1" >> /dev/rust-message-board
echo "fresh start" > /dev/rust-message-board
dd if=/dev/rust-message-board iflag=nonblock 2>/dev/null
# Prints ONLY "fresh start", while the background `cat` receives both!
kill -INT %1
```

*Finished! Compare your implementation with [Solution: Part 5](miscdev-solution-trunc.md).*
