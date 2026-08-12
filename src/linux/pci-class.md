---
minutes: 15
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Exposing Class Devices

A PCI driver operates hardware, but it must expose a **class device** (like a character device or DRM device) so userspace applications can interact with it.

---

## Option 1: Exposing via `miscdevice`

A simple character device interface (`/dev/qemu-edu`) can be added by registering a `MiscDeviceRegistration` within our PCI driver state:

```rust,ignore
#[pin_data]
struct EduMiscDevice {
    edu: Arc<EduDevice>,
}

#[pin_data(PinnedDrop)]
struct EduDriverData {
    pdev: ARef<pci::Device>,
    #[pin]
    _miscdev: MiscDeviceRegistration<EduMiscDevice>,
}

// In probe():
let options = MiscDeviceOptions {
    name: c"qemu-edu",
    parent: Some(pdev.as_ref()),
};

Ok(try_pin_init!(EduDriverData {
    pdev: pdev.into(),
    _miscdev <- MiscDeviceRegistration::register(options, edu.clone()),
}))
```

---

## Option 2: Exposing via the DRM Subsystem

For graphics cards and accelerators, the **Direct Rendering Manager (DRM)** subsystem is preferred. A DRM device (`/dev/dri/cardX`) is registered using the `drm::Driver` trait.

### Safe Lifetime Protection

Rather than accessing the PCI device dynamically (which could race with unbinding), DRM uses a sleepable **SRCU critical section** (`drm::RegistrationGuard`) to guarantee memory safety during IOCTLs:

```rust,ignore
struct EduFile;

impl drm::file::DriverFile for EduFile {
    type Driver = EduDrmDriver;
    // ...
}

impl EduFile {
    pub(crate) fn get_id(
        _dev: &drm::Device<EduDrmDriver, Registered>,
        reg_data: &EduRegistrationData<'_>,
        arg: &mut uapi::drm_edu_get_id,
        _file: &drm::File<Self>,
    ) -> Result<u32> {
        // Safe access to the BAR registers inside the SRCU read-side lock:
        arg.id = *reg_data.bar.read(regs::ID).id();
        Ok(0)
    }
}
```

- If the PCI card is unplugged or unbound, DRM prevents new IOCTL calls and safely revokes existing ones while the SRCU section finishes.

<details>

- Highlight the contrast: `miscdevice` is lightweight and simple, but has fewer built-in memory protection guarantees during dynamic unbind than the DRM subsystem's SRCU-guarded registry.
- Explain that `drm::gem::Object` handles GPU memory allocations, which are also tied into the DRM class device interface.

</details>
