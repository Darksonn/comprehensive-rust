---
minutes: 15
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Part 3: Live Streaming with `CondVar`

Turn `cat /dev/rust-message-board` into a live message stream (`tail -f` style) so readers wait for new posts instead of exiting at EOF.

## Task

Instead of returning EOF, a reader that has already read every message should sleep until a new message is posted. Writers should wake up all sleeping readers when they post a message.

**Help:** Check the documentation of [`CondVar`](https://rust.docs.kernel.org/kernel/sync/struct.CondVar.html) (especially [`wait_interruptible`](https://rust.docs.kernel.org/kernel/sync/struct.CondVar.html#method.wait_interruptible) and [`notify_all`](https://rust.docs.kernel.org/kernel/sync/struct.CondVar.html#method.notify_all)) and [`new_condvar!`](https://rust.docs.kernel.org/kernel/sync/macro.new_condvar.html). Wake-ups can be spurious, so wait in a loop, and return `ERESTARTSYS` if the wait is interrupted by a signal.

## Testing in QEMU

```bash
rmmod rust_message_board 2>/dev/null; insmod /mnt/linux/samples/rust/rust_message_board.ko
cat /dev/rust-message-board &
echo "hello live readers!" >> /dev/rust-message-board
echo "another live message!" >> /dev/rust-message-board
kill -INT %1
```

*Want to check your work before moving on to Part 4? See [Solution: Part 3](miscdev-solution-condvar.md).*
