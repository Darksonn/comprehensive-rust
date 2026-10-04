---
minutes: 30
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Exercise: Minimal DRM Driver

In this exercise, you will inspect the starter PCI DRM driver for the QEMU `edu` device ([`samples/rust/rust_driver_pci_edu_drm.rs`](https://github.com/Darksonn/linux/blob/rfl-course-edu/samples/rust/rust_driver_pci_edu_drm.rs)) and expose a dummy `GET_ID` IOCTL.

## Tasks

1. **Inspect the Starter Stub:**
   - Open [`samples/rust/rust_driver_pci_edu_drm.rs`](https://github.com/Darksonn/linux/blob/rfl-course-edu/samples/rust/rust_driver_pci_edu_drm.rs).
   - Notice that `PCI_TABLE` (`pci::Vendor::QEMU`, `0x11e8`), `EduDriver::probe`, and `EduDrmData<'drm>` (holding `bar: pci::Bar<'drm, { regs::END }>`) are already set up for you.
2. **Add the Dummy `GET_ID` IOCTL:**
   - Declare `EDU_GET_ID` in `kernel::declare_drm_ioctls!`.
   - Implement `EduFile::get_id` so that it sets `arg.id = 0x12345678` and returns `Ok(0)`.
3. **Compile and Probe in QEMU:**
   - Build just the exercise module on the host:
     ```bash
     make LLVM=1 samples/rust/rust_driver_pci_edu_drm.ko
     ```
   - Boot QEMU with `-device edu` (if not already running; see [Setup](setup.md)):
     ```bash
     cd ~/learn-rust
     qemu-system-x86_64 \
         -enable-kvm \
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
   - Inside the VM, load `samples/rust/rust_driver_pci_edu_drm.ko` and verify `/dev/dri/card0` and `/dev/dri/renderD128` appear:
     ```bash
     insmod /mnt/linux/samples/rust/rust_driver_pci_edu_drm.ko
     ls -l /dev/dri/
     ```
   - *Tip:* For subsequent exercises, you can keep QEMU running, re-run `make LLVM=1 samples/rust/rust_driver_pci_edu_drm.ko` on the host, and reload the module in the VM with `rmmod rust_driver_pci_edu_drm && insmod /mnt/linux/samples/rust/rust_driver_pci_edu_drm.ko`.
4. **Test with Userspace `test_ioctl.c`:**

```c
#include <fcntl.h>
#include <stdio.h>
#include <sys/ioctl.h>
#include <unistd.h>
#include <drm/drm.h>

struct drm_edu_get_id {
    __u32 id;
};

#define DRM_EDU_GET_ID       0x00
#define DRM_IOCTL_EDU_GET_ID DRM_IOR(DRM_COMMAND_BASE + DRM_EDU_GET_ID, struct drm_edu_get_id)

int main() {
    int fd = open("/dev/dri/renderD128", O_RDWR);
    if (fd < 0) {
        perror("Failed to open /dev/dri/renderD128");
        return 1;
    }

    struct drm_edu_get_id arg = {0};
    if (ioctl(fd, DRM_IOCTL_EDU_GET_ID, &arg) < 0) {
        perror("IOCTL failed");
        close(fd);
        return 1;
    }

    printf("Device ID: 0x%08x (expected: 0x12345678)\n", arg.id);
    close(fd);
    return 0;
}
```

```bash
gcc -static -o test_ioctl test_ioctl.c
./test_ioctl
```
