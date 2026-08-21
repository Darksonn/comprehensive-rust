---
minutes: 30
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Exercise: Device ID and Liveness

In this exercise, you will begin implementing a DRM PCI driver for the QEMU `edu` device. You will focus on mapping the MMIO BAR and implementing the `GET_ID` and `TEST_LIVENESS` IOCTLs.

---

## Hardware Information

The QEMU `edu` device has the following register specifications for this exercise:

*   **PCI Vendor ID:** `0x1234`
*   **PCI Device ID:** `0x11e8`
*   **BAR 0:** MMIO range of size `0x80` bytes.
*   **Registers:**
    *   **`ID`** (Offset `0x00`, RO, 32-bit): Returns `0x010000ed`.
    *   **`LIVENESS`** (Offset `0x04`, RW, 32-bit): Reading returns the bitwise NOT (`~`) of the last value written.

---

## Starter Code

Copy the full example from the previous page ([Memory-Mapped I/O](mmio.md)) and use it as your starting point.

### Tasks

1.  **Map BAR 0:** In `probe`, enable device memory and map BAR 0.
2.  **Define Registers:** Define `ID` and `LIVENESS` in the `register!` macro.
3.  **Implement `get_id` IOCTL:** Read the `ID` register and write it to `arg.id`.
4.  **Implement `test_liveness` IOCTL:** Write `arg.val` to the `LIVENESS` register, read it back, and write the result to `arg.inv`.

---

## Userspace Test Program

Save the following code as `test_edu.c` on your host machine:

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

#define DRM_EDU_GET_ID             0x00
#define DRM_EDU_TEST_LIVENESS      0x01

#define DRM_IOCTL_EDU_GET_ID            DRM_IOR(DRM_COMMAND_BASE + DRM_EDU_GET_ID, struct drm_edu_get_id)
#define DRM_IOCTL_EDU_TEST_LIVENESS     DRM_IOWR(DRM_COMMAND_BASE + DRM_EDU_TEST_LIVENESS, struct drm_edu_test_liveness)

void print_usage(const char *prog) {
    fprintf(stderr, "Usage:\n");
    fprintf(stderr, "  %s id              - Get device ID\n", prog);
    fprintf(stderr, "  %s live <value>    - Test liveness (writes value, expects ~value)\n", prog);
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
        printf("Device ID: 0x%08x (expected: 0x010000ed)\n", arg.id);
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

### Compiling and Running

Compile the code statically on your host machine, transfer the binary to the VM, and run it:

```bash
# Compile statically on host:
gcc -static -o test_edu test_edu.c

# Run inside the VM (assuming your driver is loaded):
./test_edu id
./test_edu live 0x12345678
```
