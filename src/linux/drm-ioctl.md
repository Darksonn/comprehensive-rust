---
minutes: 15
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Defining DRM IOCTLs

[`include/uapi/drm/qemu_edu_drm.h`](https://github.com/Darksonn/linux/blob/rfl-course-edu/include/uapi/drm/qemu_edu_drm.h):

```c
struct drm_edu_get_id {
    __u32 id;
};

#define DRM_EDU_GET_ID       0x00
#define DRM_IOCTL_EDU_GET_ID DRM_IOR(DRM_COMMAND_BASE + DRM_EDU_GET_ID, struct drm_edu_get_id)
```

In `impl drm::Driver for EduDriver`:

```rust,ignore
    kernel::declare_drm_ioctls! {
        (EDU_GET_ID, drm_edu_get_id, ioctl::RENDER_ALLOW, EduFile::get_id),
    }
```

Handler on `EduFile`:

```rust,ignore
impl EduFile {
    fn get_id(
        _dev: &drm::Device<EduDriver, Registered>,
        _reg_data: &EduDrmData<'_>,
        arg: &mut uapi::drm_edu_get_id,
        _file: &drm::File<Self>,
    ) -> Result<u32> {
        arg.id = 0x12345678;
        Ok(0)
    }
}
```

<details>

- **UAPI bindings (`kernel::uapi`):** `bindgen` processes [`rust/uapi/uapi_helper.h`](https://github.com/Darksonn/linux/blob/rfl-course-edu/rust/uapi/uapi_helper.h) (which includes [`include/uapi/drm/qemu_edu_drm.h`](https://github.com/Darksonn/linux/blob/rfl-course-edu/include/uapi/drm/qemu_edu_drm.h)) to generate `uapi::drm_edu_get_id` and `uapi::DRM_IOCTL_EDU_GET_ID`.
- **What `declare_drm_ioctls!` checks at compile time:**
  - Prepends `DRM_IOCTL_` to `EDU_GET_ID`.
  - Asserts at compile time that `size_of::<uapi::drm_edu_get_id>()` matches `_IOC_SIZE(DRM_IOCTL_EDU_GET_ID)`.
  - Copies the user-space argument into a kernel stack struct, passes `&mut uapi::drm_edu_get_id` to `EduFile::get_id`, and copies it back to user space on `Ok` (if `DRM_IOR` / `DRM_IOWR`).
- Point out `&EduDrmData<'_>`: DRM acquires its SRCU lock before calling `get_id` and passes a shared reference to the registration data.

</details>
