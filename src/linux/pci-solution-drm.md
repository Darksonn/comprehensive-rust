---
minutes: 5
---

<!--
Copyright 2026 Google LLC
 SPDX-License-Identifier: CC-BY-4.0
-->

# Solution: Minimal DRM Driver

You can find the solution code below. This code sets up the PCI DRM driver, binds to the `edu` device, and implements the dummy `GET_ID` IOCTL.

```rust,ignore
// SPDX-License-Identifier: GPL-2.0
//! Minimal DRM PCI driver with dummy IOCTL.

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

struct EduDriver;

#[pin_data]
struct EduPciData<'bound> {
    pdev: ARef<pci::Device>,
    _reg: drm::Registration<'bound, EduDriver>,
}

#[pin_data]
struct EduDrmData {}

struct EduFile;

#[pin_data]
struct EduObject {}

impl pci::Driver for EduDriver {
    type IdInfo = ();
    type Data<'bound> = EduPciData<'bound>;

    const ID_TABLE: pci::IdTable<Self::IdInfo> = &PCI_TABLE;

    fn probe<'bound>(
        pdev: &'bound pci::Device<Core<'_>>,
        _info: Option<&'bound Self::IdInfo>,
    ) -> impl PinInit<Self::Data<'bound>, Error> + 'bound {
        pin_init::pin_init_scope(move || {
            pdev.enable_device_mem()?;
            pdev.set_master();

            let unreg_dev = drm::UnregisteredDevice::<EduDriver>::new(pdev, Ok(()))?;

            // Initialize the shared DRM registration data
            let reg_data = try_pin_init!(EduDrmData {});

            let reg = unsafe {
                drm::Registration::new(pdev.as_ref(), unreg_dev, reg_data, 0)?
            };

            Ok(try_pin_init!(EduPciData {
                pdev: pdev.into(),
                _reg: reg,
            }))
        })
    }
}

#[vtable]
impl drm::Driver for EduDriver {
    type Data = ();
    type RegistrationData<'drm> = EduDrmData;
    type File = EduFile;
    type Object = drm::gem::Object<EduObject>;
    type ParentDevice<Ctx: DeviceContext> = pci::Device<Ctx>;

    const INFO: drm::DriverInfo = drm::DriverInfo {
        major: 1,
        minor: 0,
        patchlevel: 0,
        name: c"qemu_edu_drm",
        desc: c"QEMU EDU PCI DRM Driver",
    };

    const FEAT_RENDER: bool = true;

    kernel::declare_drm_ioctls! {
        (EDU_GET_ID, drm_edu_get_id, ioctl::RENDER_ALLOW, EduFile::get_id),
    }
}

impl drm::file::DriverFile for EduFile {
    type Driver = EduDriver;

    fn open(_dev: &drm::Device<EduDriver>) -> Result<Pin<KBox<Self>>> {
        Ok(KBox::new(Self, GFP_KERNEL)?.into())
    }
}

impl EduFile {
    fn get_id(
        _dev: &drm::Device<EduDriver, Registered>,
        _reg_data: &EduDrmData,
        arg: &mut uapi::drm_edu_get_id,
        _file: &drm::File<Self>,
    ) -> Result<u32> {
        // Return a dummy ID value for now
        arg.id = 0x12345678;
        Ok(0)
    }
}

impl drm::gem::DriverObject for EduObject {
    type Driver = EduDriver;
    type Args = ();

    fn new(
        _dev: &drm::Device<EduDriver>,
        _size: usize,
        _args: Self::Args,
    ) -> impl PinInit<Self, Error> {
        try_pin_init!(EduObject {})
    }
}

kernel::pci_device_table!(
    PCI_TABLE,
    <EduDriver as pci::Driver>::IdInfo,
    [(pci::DeviceId::from_id(pci::Vendor::QEMU, 0x11e8), ())]
);

kernel::module_pci_driver! {
    type: EduDriver,
    name: "rust_driver_pci_edu_drm",
    authors: ["Your Name"],
    description: "QEMU PCI EDU DRM driver (Part 1)",
    license: "GPL v2",
}
```

---

## Userspace Test Program (`test_ioctl.c`)

You can test the IOCTL using the following C program:

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
