---
minutes: 15
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Defining DRM IOCTLs

DRM drivers expose functionality to userspace via **IOCTLs (Input/Output Controls)**. In Rust, these are declared safely using the `declare_drm_ioctls!` macro, which integrates directly with the kernel's UAPI (User API) headers.

---

## The UAPI Header

A shared C UAPI header defines the interface contract between the kernel driver and userspace.

Below is a snippet of `include/uapi/drm/qemu_edu_drm.h` defining the `GET_ID` ioctl:

```c
struct drm_edu_get_id {
	__u32 id;
};

#define DRM_EDU_GET_ID             0x00

enum {
	DRM_IOCTL_EDU_GET_ID            = DRM_IOR(DRM_COMMAND_BASE + DRM_EDU_GET_ID, struct drm_edu_get_id),
};
```

---

## Example: Registering a Dummy IOCTL in Rust

When we compile the kernel, `bindgen` processes this C header, generating Rust types under `kernel::uapi`. We use the `declare_drm_ioctls!` macro to map these to our file callbacks:

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

struct TestPciDriver;

#[pin_data]
struct TestDrmData {}

// ... TestPciData, TestFile, TestObject, pci::Driver probe remain same as before ...

#[vtable]
impl drm::Driver for TestPciDriver {
    type Data = ();
    type RegistrationData<'drm> = TestDrmData;
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

    // 1. Declare the IOCTL in the driver vtable.
    // The macro automatically looks for `DRM_IOCTL_EDU_GET_ID` in `kernel::uapi`.
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

// 2. Implement the callback using a dummy value (no BAR access yet):
impl TestFile {
    fn get_id(
        _dev: &drm::Device<TestPciDriver, Registered>,
        _reg_data: &TestDrmData,
        arg: &mut uapi::drm_edu_get_id,
        _file: &drm::File<Self>,
    ) -> Result<u32> {
        // Return a dummy ID value for now
        arg.id = 0x12345678;
        Ok(0)
    }
}
```

---

## How the Macro Resolves IOCTLs

The `declare_drm_ioctls!` macro performs several safety and mapping checks under the hood:

*   **Prefix Generation:** For each entry `(cmd, struct, ...)` passed, the macro prepends `DRM_IOCTL_` to `cmd` (e.g. `EDU_GET_ID` becomes `DRM_IOCTL_EDU_GET_ID`) and imports it from `kernel::uapi::*`.
*   **Compile-time Assertions:**
    *   It asserts that the size of the Rust type (`uapi::struct`) exactly matches the size encoded in the C IOCTL command code (`_IOC_SIZE(cmd)`).
    *   It asserts that the IOCTL command numbers are sequential starting from `DRM_COMMAND_BASE` (`0x40`) without gaps.

---

## Userspace Interaction: Calling the IOCTL

To verify our DRM driver works, we can write a simple C program that opens the DRM render node and calls our `EDU_GET_ID` IOCTL.

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

- Walk through how `bindgen` converts the UAPI headers into the `uapi` module in Rust.
- Emphasize that the compile-time size check prevents mismatches between the C structure layout and the Rust structure layout (which would otherwise lead to memory corruption during ioctl copying).
- Explain that `DRM_COMMAND_BASE` is `0x40`, so `DRM_IOCTL_EDU_GET_ID` has command number `0x40`.

</details>
