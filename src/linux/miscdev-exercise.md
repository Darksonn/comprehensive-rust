---
minutes: 35
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Exercise: Shared IPC Message Board

In this exercise, you will transform the miscdevice driver into a **shared Inter-Process Communication (IPC) message board**.

Instead of allocating an isolated buffer per open file descriptor, multiple processes opening `/dev/rust_misc_device` will share the same underlying state.

## Part 1: Shared State Across Processes

1. **Define Shared State:** Create a shared struct `SharedState` holding the `Mutex<Inner>` buffer:
   ```rust,ignore
   struct Inner {
       buffer: KVVec<u8>,
   }

   #[pin_data]
   struct SharedState {
       #[pin]
       inner: Mutex<Inner>,
   }
   ```
2. **Global / Shared Registration:** Allocate and pin `SharedState` once (e.g. during module initialization or using `Arc<SharedState>`).
3. **Connect `open()`:** Update `open()` so each open file descriptor holds a clone of `Arc<SharedState>` (or a pointer to the shared state) instead of creating a new buffer.
4. **Implement `write()` and `read()`:**
   - `write()`: Append incoming bytes from `UserSliceReader` into the shared `guard.buffer` using `extend_from_slice(..., GFP_KERNEL)?`.
   - `read()`: Copy the shared message to `UserSliceWriter`. Release the lock early with `drop(guard)` before copying to user space!

## Part 2 (Stretch Goal): Process-Aware Messages

When a process writes to the device, record which process sent the message:
- Retrieve the caller PID using `kernel::task::Task::current().pid()`.
- Format the stored message to include the PID, e.g.: `"[PID 42]: <message>\n"`.

## Testing in QEMU

Rebuild the kernel, boot QEMU, and test inter-process communication:

```bash
# Process A writes a message:
echo "Hello from Process A" > /dev/rust_misc_device

# Process B reads the message:
cat /dev/rust_misc_device
# Output: Hello from Process A
```
