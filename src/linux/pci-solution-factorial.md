---
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Solution: Factorial

You can find the solution code below. This code builds on top of the **Device ID and Liveness** solution, adding register definitions and the `compute_factorial` callback using `read_poll_timeout` to sleep-poll the hardware status register.


```rust
// SPDX-License-Identifier: GPL-2.0
//! Rust PCI EDU driver sample (Part 2: Factorial).

use kernel::{
    device::{Core, DeviceContext},
    drm,
    drm::ioctl,
    drm::Registered,
    io::{poll, Io},
    pci,
    prelude::*,
    sync::aref::ARef,
    time,
    uapi,
};

mod regs {
    use kernel::io::register;
    register! {
        pub(super) ID(u32) @ 0x00 {
            31:0 id;
        }
        pub(super) LIVENESS(u32) @ 0x04 {
            31:0 val;
        }
        pub(super) FACTORIAL(u32) @ 0x08 {
            31:0 val;
        }
        pub(super) STATUS(u32) @ 0x20 {
            0:0 computing;
        }
    }
    pub(super) const END: usize = 0x80;
}

struct EduDriver;

#[pin_data]
struct EduPciData<'bound> {
    pdev: ARef<pci::Device>,
    _reg: drm::Registration<'bound, EduDriver>,
}

#[pin_data]
struct EduDrmData<'drm> {
    bar: pci::Bar<'drm, { regs::END }>,
}

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

            let bar = pdev.iomap_region_sized::<{ regs::END }>(0, c"qemu_edu_drm")?;

            let unreg_dev = drm::UnregisteredDevice::<EduDriver>::new(pdev, Ok(()))?;

            let reg_data = try_pin_init!(EduDrmData {
                bar,
            });

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
    type RegistrationData<'drm> = EduDrmData<'drm>;
    type File = EduFile;
    type Object = drm::gem::Object<EduObject>;
    type ParentDevice<Ctx: DeviceContext> = pci::Device<Ctx>;

    const INFO: drm::DriverInfo = drm::DriverInfo {
        major: 1,
        minor: 0,
        patchlevel: 0,
        name: c"qemu-edu-drm",
        desc: c"QEMU PCI EDU DRM Driver",
    };

    const FEAT_RENDER: bool = true;

    kernel::declare_drm_ioctls! {
        (EDU_GET_ID, drm_edu_get_id, ioctl::RENDER_ALLOW, EduFile::get_id),
        (EDU_TEST_LIVENESS, drm_edu_test_liveness, ioctl::RENDER_ALLOW, EduFile::test_liveness),
        (EDU_COMPUTE_FACTORIAL, drm_edu_compute_factorial, ioctl::RENDER_ALLOW, EduFile::compute_factorial),
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
        reg_data: &EduDrmData<'_>,
        arg: &mut uapi::drm_edu_get_id,
        _file: &drm::File<Self>,
    ) -> Result<u32> {
        arg.id = reg_data.bar.read(regs::ID).id().get();
        Ok(0)
    }

    fn test_liveness(
        _dev: &drm::Device<EduDriver, Registered>,
        reg_data: &EduDrmData<'_>,
        arg: &mut uapi::drm_edu_test_liveness,
        _file: &drm::File<Self>,
    ) -> Result<u32> {
        reg_data.bar.write(regs::LIVENESS, arg.val.into());
        arg.inv = reg_data.bar.read(regs::LIVENESS).val().get();
        Ok(0)
    }

    fn compute_factorial(
        _dev: &drm::Device<EduDriver, Registered>,
        reg_data: &EduDrmData<'_>,
        arg: &mut uapi::drm_edu_compute_factorial,
        _file: &drm::File<Self>,
    ) -> Result<u32> {
        let bar = &reg_data.bar;
        bar.write(regs::FACTORIAL, arg.val.into());

        poll::read_poll_timeout(
            || Ok(bar.read(regs::STATUS)),
            |status: &regs::STATUS| status.computing().get() == 0,
            time::Delta::from_millis(1),
            time::Delta::from_millis(100),
        )?;

        arg.res = bar.read(regs::FACTORIAL).val().get();
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
    description: "QEMU PCI EDU DRM driver (Part 2)",
    license: "GPL v2",
}
```

---

## Userspace Test Program (`test_edu.c`)

You can find the updated `test_edu.c` program supporting the `fact` command below:

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

#define DRM_EDU_GET_ID             0x00
#define DRM_EDU_TEST_LIVENESS      0x01
#define DRM_EDU_COMPUTE_FACTORIAL  0x02

#define DRM_IOCTL_EDU_GET_ID            DRM_IOR(DRM_COMMAND_BASE + DRM_EDU_GET_ID, struct drm_edu_get_id)
#define DRM_IOCTL_EDU_TEST_LIVENESS     DRM_IOWR(DRM_COMMAND_BASE + DRM_EDU_TEST_LIVENESS, struct drm_edu_test_liveness)
#define DRM_IOCTL_EDU_COMPUTE_FACTORIAL DRM_IOWR(DRM_COMMAND_BASE + DRM_EDU_COMPUTE_FACTORIAL, struct drm_edu_compute_factorial)

void print_usage(const char *prog) {
    fprintf(stderr, "Usage:\n");
    fprintf(stderr, "  %s id              - Get device ID\n", prog);
    fprintf(stderr, "  %s live <value>    - Test liveness (writes value, expects ~value)\n", prog);
    fprintf(stderr, "  %s fact <value>    - Compute factorial of value\n", prog);
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
