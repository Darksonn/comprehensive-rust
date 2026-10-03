---
minutes: 15
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Case Study: Credentials

[`rust/kernel/cred.rs`](https://github.com/Darksonn/linux/blob/rfl-course-edu/rust/kernel/cred.rs):

```rust,ignore
#[repr(transparent)]
pub struct Credential(Opaque<bindings::cred>);

// SAFETY: `Credential::dec_ref` can be called from any thread.
unsafe impl Send for Credential {}
// SAFETY: Shared references only access immutable or synchronized properties.
unsafe impl Sync for Credential {}

impl Credential {
    #[inline]
    pub fn euid(&self) -> Kuid {
        // SAFETY: `self.0.get()` is valid, and `euid` is immutable after creation.
        Kuid::from_raw(unsafe { (*self.0.get()).euid })
    }
}

// SAFETY: `Credential` is always ref-counted via `get_cred` / `put_cred`.
unsafe impl AlwaysRefCounted for Credential {
    fn inc_ref(&self) {
        // SAFETY: Shared reference guarantees a non-zero refcount.
        unsafe { bindings::get_cred(self.0.get()) };
    }

    unsafe fn dec_ref(obj: core::ptr::NonNull<Credential>) {
        // SAFETY: Caller guarantees a non-zero refcount; cast is valid via repr(transparent).
        unsafe { bindings::put_cred(obj.cast().as_ptr()) };
    }
}
```

<details>

- Walk through the building blocks of wrapping a C struct in `rust/kernel/`:
  1. **`#[repr(transparent)]` & `Opaque<bindings::cred>`:**
     - `Opaque<T>` tells Rust that the memory is managed by C (may contain uninitialized padding, interior mutability, or pinned fields) and provides `.get() -> *mut T`.
     - `#[repr(transparent)]` guarantees `Credential` has the exact same layout as `struct cred`, making pointer casts (`*const bindings::cred` $\leftrightarrow$ `*const Credential`) sound.
  2. **Thread safety (`Send` and `Sync`):**
     - Raw C structs do not automatically implement `Send` or `Sync`; the abstraction author explicitly opts in with `unsafe impl` and a `// SAFETY:` justification.
  3. **Safe methods (`euid`):**
     - Reading `euid` is safe without locking because `struct cred` follows RCU/copy-on-write semantics and is immutable once published.
  4. **Reference counting (`AlwaysRefCounted` $\to$ `ARef<Credential>`):**
     - Implementing `AlwaysRefCounted` hooks C's `get_cred`/`put_cred` into Rust's `ARef<Credential>` smart pointer.

</details>
