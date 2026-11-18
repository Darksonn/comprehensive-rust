---
minutes: 35
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Exercise: Shared IPC Message Board

In this exercise, you will transform [`samples/rust/rust_misc_device.rs`](https://github.com/Darksonn/linux/blob/rfl-course-edu/samples/rust/rust_misc_device.rs) into a **shared Inter-Process Communication (IPC) message board**.

Instead of allocating an isolated buffer per open file descriptor, multiple processes opening `/dev/rust-misc-device` will share the same underlying state.

## Part 1: Shared State Across Processes

1. **Define Shared State:** Move `inner: Mutex<Inner>` into a reference-counted `SharedState` struct (or store `Arc<SharedState>` inside `RustMiscDevice`):
   ```rust,ignore
   #[pin_data]
   struct SharedState {
       #[pin]
       inner: Mutex<Inner>,
   }

   #[pin_data]
   struct RustMiscDevice {
       shared: Arc<SharedState>,
   }
   ```
2. **Initialize in `init()`:** In `RustMiscDeviceModule::init`, allocate `Arc::pin_init(try_pin_init!(SharedState { ... }), GFP_KERNEL)?` and pass it when initializing `RustMiscDevice` in `MiscDeviceRegistration::register`.
3. **Connect `open()`:** Update `RustMiscDeviceFile` (or set `type File = Arc<SharedState>;`) so each open file descriptor holds a clone of `misc.shared` (`Arc::clone(&misc.shared)`) instead of creating a new `Mutex<Inner>`.
4. **Update `read_iter()`, `write_iter()`, and `ioctl()`:** Access the shared `Mutex<Inner>` through `shared.inner.lock()`.

## Part 2 (Stretch Goal): Process-Aware Messages

When a process writes to the device, record which process sent the message:
- Retrieve the caller PID using `current!().pid()`.
- Format the stored message to include the PID, e.g.: `"[PID 42]: <message>\n"`.

## Testing in QEMU

Rebuild `samples/rust/rust_misc_device.ko`, load it in QEMU, and test inter-process communication:

```bash
# Process A writes a message:
echo "Hello from Process A" > /dev/rust-misc-device

# Process B reads the message:
cat /dev/rust-misc-device
# Output: Hello from Process A
```
