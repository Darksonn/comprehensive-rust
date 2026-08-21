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

## Booting QEMU with the EDU Device

To run the VM with the virtual PCI card inserted, append the `-device edu` option to the QEMU command:

```bash
cd ~/learn-rust
qemu-system-x86_64 \
    -machine q35,acpi=on \
    -kernel linux/arch/x86/boot/bzImage \
    -drive file=debian.img,format=raw,if=virtio \
    -append "root=/dev/vda console=ttyS0 acpi=force net.ifnames=0" \
    -nographic \
    -no-reboot \
    -m 2G -smp 2 \
    -virtfs local,path=$PWD,mount_tag=hostshare,security_model=none,id=hostshare \
    -device edu
```

Once booted, verify that the device is detected on the PCIe bus:

```bash
lspci -d 1234:11e8 -v
```

If `lspci` is not available, you can verify via `sysfs`:

```bash
grep -H 0x11e8 /sys/bus/pci/devices/*/device
```

---

## Hardware Register Map

The device registers are located at the following offsets inside **BAR 0** (size `0x80` bytes). The bolded registers are the ones introduced in this exercise:

| Register | Offset | Access | Width | Description |
| :--- | :---: | :---: | :---: | :--- |
| `ID` | `0x00` | RO | 32-bit | Returns identification register: `0x010000ed`. |
| `LIVENESS` | `0x04` | RW | 32-bit | Writing value `X` returns `~X` when read back. |
| `FACTORIAL` | `0x08` | RW | 32-bit | Write `N` to start computing `N!`. Read it back to get the result. |
| `STATUS` | `0x20` | RO | 32-bit | Bit 0: `1` if computing factorial, `0` if idle.<br>Bit 7: `1` if an interrupt is raised. |
| **`IRQ_STATUS`** | `0x24` | RO | 32-bit | Read pending interrupt value. |
| **`IRQ_ACKNOWLEDGE`** | `0x64` | WO | 32-bit | Write the pending interrupt value back to clear/acknowledge it. |
| **`IRQ_RAISE`** | `0x60` | WO | 32-bit | Write any value here to raise an interrupt (MSI) for testing. |

---

## Tasks

### 1. Refactor to Share the BAR with the Interrupt Handler

Previously, you stored the `Bar` directly in `EduDrmData`. Because the interrupt handler also needs to access the BAR, you must move it:

1.  Define the `EduIrqHandler` struct to hold the `Bar`:
    ```rust
    #[pin_data]
    struct EduIrqHandler<'bound> {
        pdev: &'bound pci::Device<Bound>,
        bar: pci::Bar<'bound, { regs::END }>,
    }
    ```
2.  Update `EduDrmData` to hold the interrupt registration instead of the BAR:
    ```rust
    #[pin_data]
    struct EduDrmData<'drm> {
        #[pin]
        _irq: irq::Registration<'drm, EduIrqHandler<'drm>>,
    }
    ```
3.  Update your IOCTL callbacks (`get_id`, `test_liveness`, `compute_factorial`) to access the BAR via the interrupt handler:
    ```rust
    let bar = &reg_data._irq.handler().bar;
    ```

### 2. Implement the Interrupt Handler

Implement the `irq::Handler` trait for `EduIrqHandler`:
1.  Read the pending interrupt status from `IRQ_STATUS`.
2.  If the status is `0`, return `irq::IrqReturn::None` (it wasn't our interrupt).
3.  Log a message using `dev_info!`.
4.  Write the status value back to `IRQ_ACKNOWLEDGE` to acknowledge it.
5.  Return `irq::IrqReturn::Handled`.

### 3. Request IRQ in `probe`

In your `probe` function, allocate and request the interrupt:
1.  Allocate 1 MSI vector using `pdev.alloc_irq_vectors(1, 1, pci::IrqTypes::all())?`.
2.  Get the vector number using `*vectors.start()`.
3.  Request the IRQ using `pdev.request_irq`. Pass the vector, `irq::Flags::SHARED`, and an initialized `EduIrqHandler`.
    *   *Note:* `request_irq` is `unsafe` because you must guarantee that the handler is valid as long as the IRQ is registered. In our case, the registration is stored in `EduDrmData` and is dropped when the DRM device is unregistered, which happens before the PCI device is unbound. Write a safety comment explaining this.
4.  Pass the `irq_init` registration to `EduDrmData`.

### 4. Implement `test_irq` IOCTL

Implement the `test_irq` callback:
1.  Write `arg.val` to the `IRQ_RAISE` register to trigger a hardware interrupt.
2.  Register the `EDU_TEST_IRQ` IOCTL in `declare_drm_ioctls!` mapping to this callback.

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
