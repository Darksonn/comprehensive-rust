---
minutes: 20
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Exercise: Factorial

In this exercise, you will add the factorial computation feature to your QEMU `edu` driver. You will build on top of your solution from the **Device ID and Liveness** exercise.

---

## Hardware Information

The QEMU `edu` device has the following register specifications for this exercise:

*   **`FACTORIAL`** (Offset `0x08`, RW, 32-bit): Write `N` to start computing `N!`. Read it back to get the result.
*   **`STATUS`** (Offset `0x20`, RO, 32-bit):
    *   Bit 0: `1` if computing factorial, `0` if idle.

---

## Tasks

1.  **Define Registers:** Add `FACTORIAL` and `STATUS` (with the `computing` bit at bit 0) to your `register!` macro.
2.  **Implement `compute_factorial` IOCTL:**
    *   Write `arg.val` to the `FACTORIAL` register.
    *   Use `read_poll_timeout` to poll the `STATUS` register until the `computing` bit becomes `0` (idle).
    *   Read the result from the `FACTORIAL` register and write it to `arg.res`.
3.  **Register the IOCTL:** Add the `EDU_COMPUTE_FACTORIAL` IOCTL to `declare_drm_ioctls!` and map it to your callback.
4.  **Synchronize Access (Stretch Goal):** If multiple users attempt to compute factorials simultaneously, they will interfere with each other's register writes. Synchronize access to the factorial registers with a Mutex to prevent this.

---

## Testing

Update your `test_edu.c` program on your host machine to support the new `fact` command (similar to how `live` was implemented), recompile it statically, and copy it to the VM.

Then test the new functionality:

```bash
# Compute factorial of 5 (expects 120):
./test_edu fact 5

# Compute factorial of 10 (expects 3628800):
./test_edu fact 10
```
