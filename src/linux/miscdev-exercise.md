---
minutes: 35
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Exercise: Kernel Message Board

Starting from [`samples/rust/rust_message_board.rs`](https://github.com/Darksonn/linux/blob/rfl-course-edu/samples/rust/rust_message_board.rs) (or the `rust_misc_device.rs` we just updated together), turn the shared single-buffer device into a **live multi-reader message board** (`/dev/rust-message-board`).

## Part 1: Multi-Message Log & Per-File Cursor

1. **Store a List of Messages:** Replace `buffer: KVVec<u8>` in `Inner` with a vector of messages:
   ```rust,ignore
   struct Inner {
       messages: KVec<KVec<u8>>,
   }
   ```
2. **Track a Per-File Cursor:** Add `#[pin] cursor: Mutex<usize>` (initialized to `0` via `new_mutex!(0)`) to `RustMessageBoardFile` so each open file descriptor remembers the index of the next message it should read.
3. **Append Messages in `write_iter()`:**
   - Ignore empty writes (`if iov.is_empty() { return Ok(0); }`) so readers don't mistake a 0-byte message for EOF.
   - Copy incoming bytes into a new `KVec<u8>` using `iov.copy_from_iter_vec(&mut msg, GFP_KERNEL)?` and `push` it onto `inner.messages`.
   - *Optional:* Prepend the sender's PID using `CString::try_from_fmt(fmt!("[PID {}]: ", current!().pid()))?` and `msg.extend_from_slice(prefix.to_bytes(), GFP_KERNEL)?` before appending the user payload!
4. **Read One Message at a Time in `read_iter()`:**
   - Lock `me.cursor` and `me.shared.inner`.
   - If `*cursor >= inner.messages.len()`, return `Ok(0)` (EOF).
   - Otherwise, copy `&inner.messages[*cursor]` to user space with `iov.copy_to_iter(...)`, increment `*cursor += 1`, and return the number of bytes written.

Build and test Part 1 in QEMU:

```bash
# On the host:
make LLVM=1 samples/rust/rust_message_board.ko

# Inside QEMU:
insmod /mnt/linux/samples/rust/rust_message_board.ko
echo "First post!" > /dev/rust-message-board
echo "Second post!" > /dev/rust-message-board
cat /dev/rust-message-board
# Prints both messages, and a second `cat` prints the full log again!
```

## Part 2: Live Streaming Reads with `CondVar`

Make `cat /dev/rust-message-board` behave like a live chat stream (`tail -f`) instead of exiting at EOF:

1. **Add a `CondVar`:** Add `#[pin] condvar: CondVar` to `SharedState` (initialized with `condvar <- new_condvar!()`).
2. **Wake Readers on Write:** In `write_iter()`, call `me.shared.condvar.notify_all()` after pushing the new message.
3. **Sleep Until a Message Arrives:** In `read_iter()`, replace the EOF early return with a `while` loop:
   ```rust,ignore
   while *cursor >= inner.messages.len() {
       if me.shared.condvar.wait_interruptible(&mut inner) {
           return Err(ERESTARTSYS);
       }
   }
   ```

Reload the module in QEMU and test live streaming:

```bash
rmmod rust_message_board && insmod /mnt/linux/samples/rust/rust_message_board.ko
cat /dev/rust-message-board &
echo "Hello live readers!" > /dev/rust-message-board
echo "Another message!" > /dev/rust-message-board
```
