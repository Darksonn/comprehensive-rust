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

### 1. Implementing the `drm::Driver` Trait

The DRM driver defines the file operations class, GEM object type, parent device class (PCI in our case), and supported IOCTLs.

```rust,ignore
struct EduDrmDriver;

#[vtable]
impl drm::Driver for EduDrmDriver {
    type Data = ();
    type RegistrationData<'drm> = EduRegistrationData<'drm>;
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
    }
}
```

### 2. File Operations and Safe Lifetime Protection

Rather than accessing the PCI device dynamically (which could race with unbinding), DRM uses a sleepable **SRCU critical section** (`drm::RegistrationGuard`) to guarantee memory safety during IOCTLs. The context is passed via `EduRegistrationData`:

```rust,ignore
struct EduFile;

impl drm::file::DriverFile for EduFile {
    type Driver = EduDrmDriver;

    fn open(_dev: &drm::Device<EduDrmDriver>) -> Result<Pin<KBox<Self>>> {
        Ok(KBox::new(Self, GFP_KERNEL)?.into())
    }
}

impl EduFile {
    pub(crate) fn get_id(
        _dev: &drm::Device<EduDrmDriver, Registered>,
        reg_data: &EduRegistrationData<'_>,
        arg: &mut uapi::drm_edu_get_id,
        _file: &drm::File<Self>,
    ) -> Result<u32> {
        // Safe access to the BAR registers via RegistrationData:
        let bar = &reg_data._irq.handler().bar;
        arg.id = *bar.read(regs::ID).id();
        Ok(0)
    }
}
```

### 3. Registering the DRM Device in `probe()`

We register the DRM device inside the PCI `probe()` hook. The returned `drm::Registration` takes ownership of the registration data (which also nests the IRQ handler registration):

```rust,ignore
// Inside pci::Driver::probe():
pin_init::pin_init_scope(move || {
    // ... MMIO and IRQ setup ...

    // 1. Create the unregistered DRM device instance:
    let unreg_dev = drm::UnregisteredDevice::<EduDrmDriver>::new(probe_pdev, Ok(()))?;

    // 2. Prepare the registration data containing the IRQ registration:
    let reg_data = try_pin_init!(EduRegistrationData {
        pdev: &**probe_pdev,
        _irq <- irq_init,
    });

    // 3. Register the DRM device with the kernel (exposed to userspace):
    // SAFETY: `_reg` is stored in `EduDriverData` and dropped on unbind.
    let _reg = unsafe {
        drm::Registration::new(probe_pdev.as_ref(), unreg_dev, reg_data, 0)?
    };

    Ok(try_pin_init!(EduDriverData {
        pdev: probe_pdev.into(),
        _reg,
    }))
})
```

- If the PCI card is unplugged or unbound, DRM prevents new IOCTL calls and safely revokes existing ones while the SRCU section finishes. The drop sequence automatically unregisters the IRQ handler before the BAR memory mapping is torn down.

<details>

- Highlight the contrast: `miscdevice` is lightweight and simple, but has fewer built-in memory protection guarantees during dynamic unbind than the DRM subsystem's SRCU-guarded registry.
- Explain that `drm::gem::Object` handles GPU memory allocations, which are also tied into the DRM class device interface.

</details>
