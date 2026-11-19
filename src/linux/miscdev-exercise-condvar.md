---
minutes: 15
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Part 3: Live Streaming with `CondVar`

Turn `cat /dev/rust-message-board` into a live message stream (`tail -f` style) so readers wait for new posts instead of exiting at EOF.

## Tasks

1. **Add a `CondVar` to `SharedState`:**
   ```rust,ignore
   #[pin_data]
   struct SharedState {
       #[pin]
       inner: Mutex<Inner>,
       #[pin]
       condvar: CondVar,
   }
   ```
   Initialize it in `init()` with `condvar <- new_condvar!()`.
2. **Wake Readers on Write:** In `write_iter()`, drop the `inner` lock after `push`ing `msg` and call `me.shared.condvar.notify_all()`.
3. **Sleep Until a Message Arrives:** In `read_iter()`, make `inner` mutable and replace the `if *cursor >= inner.messages.len()` early return with:
   ```rust,ignore
   while *cursor >= inner.messages.len() {
       if me.shared.condvar.wait_interruptible(&mut inner) {
           return Err(ERESTARTSYS);
       }
   }
   ```

## Testing in QEMU

```bash
rmmod rust_message_board && insmod /mnt/linux/samples/rust/rust_message_board.ko
cat /dev/rust-message-board &
echo "hello live readers!" >> /dev/rust-message-board
echo "another live message!" >> /dev/rust-message-board
kill -INT %1
```

*Want to check your work before moving on to Part 4? See [Solution: Part 3](miscdev-solution-condvar.md).*
