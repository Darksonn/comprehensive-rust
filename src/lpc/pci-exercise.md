---
minutes: 50
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Exercise: Interrupts

In this exercise, you will complete the PCI DRM driver for the QEMU `edu` device by implementing MSI interrupt handling. You will build on top of your solution from the **Factorial** exercise.

---

## Hardware Information

The QEMU `edu` device has the following register specifications for interrupts:

*   **`IRQ_STATUS`** (Offset `0x24`, RO, 32-bit): Read pending interrupt status.
*   **`IRQ_RAISE`** (Offset `0x60`, WO, 32-bit): Write any value here to raise an interrupt (MSI) for testing.
*   **`IRQ_ACKNOWLEDGE`** (Offset `0x64`, WO, 32-bit): Write the pending interrupt value back to clear/acknowledge it.
*   **`STATUS`** (Offset `0x20`, RO, 32-bit):
    *   Bit 7: `1` if an interrupt is raised.

---

## Tasks

1.  **Define Registers:** Add `IRQ_STATUS`, `IRQ_RAISE`, and `IRQ_ACKNOWLEDGE` (and optionally bit 7 `irq` on `STATUS`) to your `register!` macro inside `mod regs`.
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

## Updating the Userspace Test Program

To test the new factorial and interrupt functionality, update your `test_edu.c` program on your host machine to support the new IOCTLs:

```c
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/ioctl.h>
#include <unistd.h>
#include <drm/drm.h>

struct drm_edu_get_id {
    __u32 id;
};

struct drm_edu_test_liveness {
    __u32 val;
    __u32 inv;
};

struct drm_edu_compute_factorial {
    __u32 val;
    __u32 res;
};

struct drm_edu_test_irq {
    __u32 val;
};

#define DRM_EDU_GET_ID             0x00
#define DRM_EDU_TEST_LIVENESS      0x01
#define DRM_EDU_COMPUTE_FACTORIAL  0x02
#define DRM_EDU_TEST_IRQ           0x03

#define DRM_IOCTL_EDU_GET_ID            DRM_IOR(DRM_COMMAND_BASE + DRM_EDU_GET_ID, struct drm_edu_get_id)
#define DRM_IOCTL_EDU_TEST_LIVENESS     DRM_IOWR(DRM_COMMAND_BASE + DRM_EDU_TEST_LIVENESS, struct drm_edu_test_liveness)
#define DRM_IOCTL_EDU_COMPUTE_FACTORIAL DRM_IOWR(DRM_COMMAND_BASE + DRM_EDU_COMPUTE_FACTORIAL, struct drm_edu_compute_factorial)
#define DRM_IOCTL_EDU_TEST_IRQ          DRM_IOW(DRM_COMMAND_BASE + DRM_EDU_TEST_IRQ, struct drm_edu_test_irq)

void print_usage(const char *prog) {
    fprintf(stderr, "Usage:\n");
    fprintf(stderr, "  %s id              - Get device ID\n", prog);
    fprintf(stderr, "  %s live <value>    - Test liveness (writes value, expects ~value)\n", prog);
    fprintf(stderr, "  %s fact <value>    - Compute factorial of value\n", prog);
    fprintf(stderr, "  %s irq <value>     - Trigger interrupt with value\n", prog);
}

int main(int argc, char *argv[]) {
    if (argc < 2) {
        print_usage(argv[0]);
        return 1;
    }

    int fd = open("/dev/dri/renderD128", O_RDWR);
    if (fd < 0) {
        perror("Failed to open /dev/dri/renderD128");
        return 1;
    }

    const char *cmd = argv[1];

    if (strcmp(cmd, "id") == 0) {
        struct drm_edu_get_id arg = {0};
        if (ioctl(fd, DRM_IOCTL_EDU_GET_ID, &arg) < 0) {
            perror("GET_ID failed");
            close(fd);
            return 1;
        }
        printf("Device ID: 0x%08x\n", arg.id);
    } else if (strcmp(cmd, "live") == 0) {
        if (argc < 3) {
            fprintf(stderr, "Error: 'live' requires an integer argument.\n");
            print_usage(argv[0]);
            close(fd);
            return 1;
        }
        unsigned int val = strtoul(argv[2], NULL, 0);
        struct drm_edu_test_liveness arg = { .val = val };
        if (ioctl(fd, DRM_IOCTL_EDU_TEST_LIVENESS, &arg) < 0) {
            perror("LIVENESS failed");
            close(fd);
            return 1;
        }
        printf("Liveness: written=0x%08x, read=0x%08x (expected: 0x%08x)\n",
               val, arg.inv, ~val);
    } else if (strcmp(cmd, "fact") == 0) {
        if (argc < 3) {
            fprintf(stderr, "Error: 'fact' requires an integer argument.\n");
            print_usage(argv[0]);
            close(fd);
            return 1;
        }
        unsigned int val = strtoul(argv[2], NULL, 0);
        struct drm_edu_compute_factorial arg = { .val = val };
        if (ioctl(fd, DRM_IOCTL_EDU_COMPUTE_FACTORIAL, &arg) < 0) {
            perror("FACTORIAL failed");
            close(fd);
            return 1;
        }
        printf("Factorial: %u! = %u\n", val, arg.res);
    } else if (strcmp(cmd, "irq") == 0) {
        if (argc < 3) {
            fprintf(stderr, "Error: 'irq' requires an integer argument.\n");
            print_usage(argv[0]);
            close(fd);
            return 1;
        }
        unsigned int val = strtoul(argv[2], NULL, 0);
        struct drm_edu_test_irq arg = { .val = val };
        if (ioctl(fd, DRM_IOCTL_EDU_TEST_IRQ, &arg) < 0) {
            perror("IRQ failed");
            close(fd);
            return 1;
        }
        printf("IRQ triggered with value %u. Check dmesg for handled log.\n", val);
    } else {
        fprintf(stderr, "Error: Unknown command '%s'\n", cmd);
        print_usage(argv[0]);
        close(fd);
        return 1;
    }

    close(fd);
    return 0;
}
```

Compile it statically on your host machine and run it inside the VM:

```bash
# Compile on host:
gcc -static -o test_edu test_edu.c

# Run inside the VM (assuming your driver is loaded):
./test_edu fact 5
./test_edu irq 42

# Check dmesg to see if the interrupt was handled:
dmesg | tail
```
