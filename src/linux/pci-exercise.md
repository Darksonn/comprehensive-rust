---
minutes: 30
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Exercise: Interrupts

In this final exercise, you will add interrupt handling to your QEMU `edu` driver. You will build on top of your solution from the **Factorial** exercise.

---

## Hardware Information

The QEMU `edu` device has the following register specifications for interrupts:

*   **`IRQ_STATUS`** (Offset `0x24`, RO, 32-bit): Read pending interrupt status.
*   **`IRQ_RAISE`** (Offset `0x60`, WO, 32-bit): Write any value here to raise an interrupt (MSI) for testing.
*   **`IRQ_ACKNOWLEDGE`** (Offset `0x64`, WO, 32-bit): Write the pending interrupt value back to clear/acknowledge it.
*   **`STATUS`** (Offset `0x20`, RW, 32-bit):
    *   Bit 7 (`0x80`): Set to `1` to raise an interrupt after finishing factorial computation.

---

## Tasks

1.  **Define Registers:** Add `IRQ_STATUS: u32 @ 0x24;`, `IRQ_RAISE: u32 @ 0x60;`, and `IRQ_ACKNOWLEDGE: u32 @ 0x64;` (and optionally bit 7 `irq` on `STATUS`) to your `register!` macro inside `mod regs`.
2.  **Define `EduIrqHandler` and Update `EduDrmData`:** Create a new `EduIrqHandler` struct that borrows `bar` (with field lifetime `'bar`) and `pdev`:
    ```rust,ignore
    #[pin_data]
    struct EduIrqHandler<'bar, 'bound> {
        pdev: &'bound pci::Device<Bound>,
        bar: &'bar pci::Bar<'bound, { regs::END }>,
    }
    ```
    Update `EduDrmData` to store `_irq` and `vectors` before `bar` (so `_irq` is dropped before `vectors` and `bar`):
    ```rust,ignore
    #[pin_data]
    struct EduDrmData<'drm> {
        #[pin]
        _irq: irq::Registration<'vectors, EduIrqHandler<'bar, 'drm>>,
        vectors: pci::IrqVectorRegistration<'drm>,
        bar: pci::Bar<'drm, { regs::END }>,
    }
    ```
3.  **Implement the Interrupt Handler:** Implement the `irq::Handler` trait for `EduIrqHandler<'_, '_>`:
    *   Read the pending interrupt status from `IRQ_STATUS`.
    *   If the status is `0`, return `irq::IrqReturn::None` (it wasn't our interrupt).
    *   Log a message using `dev_info!`.
    *   Write the status value back to `IRQ_ACKNOWLEDGE` to acknowledge it.
    *   Return `irq::IrqReturn::Handled`.
4.  **Request IRQ in `probe`:** In your `probe` function, initialize `vectors` and `_irq` inside `try_pin_init!(EduDrmData { ... })`:
    *   Initialize `vectors: pdev.alloc_irq_vectors(1, 1, pci::IrqTypes::all())?`.
    *   Initialize `_irq` using `irq::Registration::new(vectors.index(0)?.into(), irq::Flags::SHARED, c"qemu_edu_drm", try_pin_init!(EduIrqHandler { pdev, bar }))`.
    *   *Note:* `irq::Registration::new` is `unsafe` because you must guarantee that the registration is not leaked (`mem::forget`) and is dropped while the device is still bound. Write a safety comment explaining this.
5.  **Implement `test_irq` IOCTL:**
    *   Write `arg.val` to the `IRQ_RAISE` register to trigger a hardware interrupt.
    *   Register the `EDU_TEST_IRQ` IOCTL in `declare_drm_ioctls!` mapping to this callback.

---

## Testing

Update your `~/learn-rust/test_edu.c` program on your host machine to support the new `irq` command (see [`include/uapi/drm/qemu_edu_drm.h`](https://github.com/Darksonn/linux/blob/rfl-course-edu/include/uapi/drm/qemu_edu_drm.h)), recompile it statically on the host, and run it via `/mnt` inside QEMU:

```c
struct drm_edu_test_irq {
    __u32 val;
};

#define DRM_EDU_TEST_IRQ  0x03
#define DRM_IOCTL_EDU_TEST_IRQ DRM_IOW(DRM_COMMAND_BASE + DRM_EDU_TEST_IRQ, struct drm_edu_test_irq)
```

Then compile on the host and test the interrupt handling inside QEMU:

```bash
# On the host, in ~/learn-rust:
gcc -static -o test_edu test_edu.c

# Inside QEMU — trigger an interrupt with value 0x42:
/mnt/test_edu irq 0x42

# Check dmesg to verify the interrupt handler was executed:
dmesg | tail -n 5
```
