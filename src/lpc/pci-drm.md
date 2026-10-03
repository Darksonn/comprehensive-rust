---
minutes: 10
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Exposing via the DRM Subsystem

```rust,ignore
struct EduDriver;

#[pin_data(PinnedDrop)]
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

#[vtable]
impl drm::Driver for EduDriver {
    type Data = ();
    type RegistrationData<'drm> = EduDrmData<'drm>;
    type File = EduFile;
    type Object = drm::gem::Object<EduObject>;
    type ParentDevice<Ctx: DeviceContext> = pci::Device<Ctx>;

    const INFO: drm::DriverInfo = drm::DriverInfo {
        major: 1, minor: 0, patchlevel: 0,
        name: c"qemu-edu-drm",
        desc: c"QEMU PCI EDU DRM Driver",
    };
    const FEAT_RENDER: bool = true;

    kernel::declare_drm_ioctls! {
        // Declare DRM IOCTLs here.
    }
}
```

<details>

- Walk through how `pci::Driver` and `drm::Driver` connect (already wired up in [`samples/rust/rust_driver_pci_edu_drm.rs`](https://github.com/Darksonn/linux/blob/rfl-course-edu/samples/rust/rust_driver_pci_edu_drm.rs)):
  - `EduPciData<'bound>` is returned by `pci::Driver::probe` and holds `_reg: drm::Registration<'bound, EduDriver>`. When the PCI device unbinds, `_reg` is dropped, which automatically unregisters the DRM device and waits for any in-flight user-space calls to complete.
  - `RegistrationData<'drm>` (`EduDrmData<'drm>`) holds resources tied to the bound hardware lifetime (`'drm` / `'bound`), such as the mapped MMIO `bar: pci::Bar<'drm, { regs::END }>`.
  - `File` (`EduFile`) represents per-`open()` file state, and `Object` (`drm::gem::Object<EduObject>`) represents GEM buffer objects.
  - Note: `pci::Driver::probe` (which maps `bar` and creates `drm::Registration`) is already implemented for you in [`samples/rust/rust_driver_pci_edu_drm.rs`](https://github.com/Darksonn/linux/blob/rfl-course-edu/samples/rust/rust_driver_pci_edu_drm.rs).

</details>
