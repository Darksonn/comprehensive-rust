---
minutes: 15
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Part 2: Discrete Messages & Per-File Cursor

A single flat `KVVec<u8>` loses message boundaries. Replace it with a list of discrete messages and track which message each open file descriptor should read next.

## Task

Store each write as a separate message instead of appending to one flat buffer, and give each open file its own read position. Each `read()` should return exactly one whole message (the next one that file has not yet read), and return EOF (`0`) once that file has read every message.

**Help:** Check the documentation of [`KVec`](https://rust.docs.kernel.org/kernel/alloc/kvec/type.KVec.html) and [`IovIterDest::copy_to_iter`](https://rust.docs.kernel.org/kernel/iov/struct.IovIterDest.html#method.copy_to_iter). Because `kiocb.file()` returns a shared reference (`Pin<&RustMessageBoardFile>`), a per-file cursor needs interior mutability such as a `#[pin] Mutex<usize>` initialized with `new_mutex!(0)` in `open()`.

## Testing in QEMU

```bash
rmmod rust_message_board 2>/dev/null; insmod /mnt/linux/samples/rust/rust_message_board.ko
echo "first message" >> /dev/rust-message-board
echo "second message" >> /dev/rust-message-board
cat /dev/rust-message-board

# Verify that one open fd returns one discrete message per read():
exec 3< /dev/rust-message-board
dd bs=1024 count=1 <&3 2>/dev/null   # Output: first message
dd bs=1024 count=1 <&3 2>/dev/null   # Output: second message
dd bs=1024 count=1 <&3 2>/dev/null   # Output: (EOF)
exec 3<&-
```

*Want to check your work before moving on to Part 3? See [Solution: Part 2](miscdev-solution-cursor.md).*
