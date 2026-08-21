---
minutes: 15
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Defining DRM IOCTLs

DRM drivers expose functionality to userspace via **IOCTLs (Input/Output Controls)**. In Rust, these are declared safely using the `declare_drm_ioctls!` macro.

*   **Automatic Copying:** The Rust DRM abstraction automatically handles copying arguments from userspace into kernel memory before the callback, and copying them back after the callback succeeds. No manual `copy_from_user` or `copy_to_user` is required.
*   **Safe Callbacks:** IOCTL callbacks are type-safe and receive a mutable reference to the unpacked argument struct.

---

## Example: Registering a Dummy IOCTL

Below, we extend our minimal DRM driver to expose the `GET_ID` IOCTL, returning a hardcoded dummy value.

```rust,ignore
use kernel::{
    device::{Core, DeviceContext},
    drm,
    drm::ioctl,
    drm::Registered,
    pci,
    prelude::*,
    sync::aref::ARef,
    uapi,
};

// Reusing the edu DRM UAPI cmd identifier for testing
#[allow(dead_code)]
const EDU_GET_ID: u32 = kernel::ioctl::_IOR::<u32>('E' as u32, 0x00);

struct TestPciDriver;

// ... TestPciData, TestFile, TestObject, pci::Driver probe remain same as before ...

#[vtable]
impl drm::Driver for TestPciDriver {
    type Data = ();
    type RegistrationData<'drm> = ();
    type File = TestFile;
    type Object = drm::gem::Object<TestObject>;
    type ParentDevice<Ctx: DeviceContext> = pci::Device<Ctx>;

    const INFO: drm::DriverInfo = drm::DriverInfo {
        major: 1,
        minor: 0,
        patchlevel: 0,
        name: c"pci-testdev-drm",
        desc: c"PCI Testdev DRM Driver",
    };

    const FEAT_RENDER: bool = true;

    // 1. Declare the IOCTL in the driver vtable:
    kernel::declare_drm_ioctls! {
        (EDU_GET_ID, drm_edu_get_id, ioctl::RENDER_ALLOW, TestFile::get_id),
    }
}

impl drm::file::DriverFile for TestFile {
    type Driver = TestPciDriver;

    fn open(_dev: &drm::Device<TestPciDriver>) -> Result<Pin<KBox<Self>>> {
        Ok(KBox::new(Self, GFP_KERNEL)?.into())
    }
}

// 2. Implement the callback:
impl TestFile {
    fn get_id(
        _dev: &drm::Device<TestPciDriver, Registered>,
        _reg_data: &(), // Shared registration data is empty for now
        arg: &mut uapi::drm_edu_get_id,
        _file: &drm::File<Self>,
    ) -> Result<u32> {
        // Return a dummy ID value
        arg.id = 0x12345678;
        Ok(0)
    }
}
```

---

## Userspace Interaction: Calling the IOCTL

To verify our DRM driver works and we can call the IOCTL from userspace, we can write a simple C program that opens the DRM render node and calls our `EDU_GET_ID` IOCTL.

Save the following code as `test_ioctl.c`:

```c
#include <fcntl.h>
#include <stdio.h>
#include <sys/ioctl.h>
#include <unistd.h>
#include <drm/drm.h>

struct drm_edu_get_id {
    __u32 id;
};

#define DRM_EDU_GET_ID             0x00
#define DRM_IOCTL_EDU_GET_ID       DRM_IOR(DRM_COMMAND_BASE + DRM_EDU_GET_ID, struct drm_edu_get_id)

int main() {
    // Open the render node (does not require root/master privileges)
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

### Compiling and Running

You can compile this program on your host machine statically, copy it to the virtual machine, and run it.

```bash
# Compile statically on host:
gcc -static -o test_ioctl test_ioctl.c

# Run inside the VM (assuming your driver is loaded):
./test_ioctl
```

<details>

- Explain the arguments of `declare_drm_ioctls!`:
  - `EDU_GET_ID`: The IOCTL code.
  - `drm_edu_get_id`: The UAPI struct name (the macro resolves this to `uapi::drm_edu_get_id`).
  - `ioctl::RENDER_ALLOW`: Permission flags (allows calling from render nodes).
  - `TestFile::get_id`: The handler function.
- Point out that the handler function must return a `Result<u32>`, where `0` indicates success.
- Mention that the `_reg_data` argument allows the handler to access shared driver state (like mapped BARs) which we will cover next.

</details>
