---
minutes: 15
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Exercise: Discrete Messages & Per-File Cursor

A single flat `KVVec<u8>` loses message boundaries. Replace it with a list of discrete messages and track which message each open file descriptor should read next.

## Tasks

1. **Store a Message List:** Replace `buffer: KVVec<u8>` in `Inner` with:
   ```rust,ignore
   struct Inner {
       messages: KVec<KVec<u8>>,
   }
   ```
2. **Track a Per-File Cursor:** Add `#[pin] cursor: Mutex<usize>` (initialized via `cursor <- new_mutex!(0)`) to `RustMessageBoardFile`.
3. **Push New Messages in `write_iter()`:**
   - Return `Ok(0)` early if `iov.is_empty()`.
   - Allocate a fresh `let mut msg = KVec::new();`, copy user bytes with `let len = iov.copy_from_iter_vec(&mut msg, GFP_KERNEL)?;`, and if `len > 0`, push `msg` with `inner.messages.push(msg, GFP_KERNEL)?;`.
4. **Read One Message Per Call in `read_iter()`:**
   - Lock `me.cursor` and `me.shared.inner` (remove `mut` from `kiocb`).
   - If `*cursor >= inner.messages.len()`, return `Ok(0)` (EOF).
   - Otherwise call `let written = iov.copy_to_iter(&inner.messages[*cursor]);`, increment `*cursor += 1`, and return `Ok(written)`.

## Testing in QEMU

```bash
rmmod rust_message_board && insmod /mnt/linux/samples/rust/rust_message_board.ko
echo "first message" > /dev/rust-message-board
echo "second message" > /dev/rust-message-board
cat /dev/rust-message-board

# Verify that one open fd returns one discrete message per read():
exec 3< /dev/rust-message-board
dd bs=1024 count=1 <&3 2>/dev/null   # Output: first message
dd bs=1024 count=1 <&3 2>/dev/null   # Output: second message
dd bs=1024 count=1 <&3 2>/dev/null   # Output: (EOF)
exec 3<&-
```
