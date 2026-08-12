---
minutes: 15
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Case Study: Credentials Abstraction

To see how safe abstractions are written, let's examine the kernel's credential wrapper: **`rust/kernel/cred.rs`**.

```rust,ignore
/// Wraps the kernel's `struct cred`.
#[repr(transparent)]
pub struct Credential(Opaque<bindings::cred>);
```

## 1. Struct Layout & `Opaque<T>`

- **`Opaque<bindings::cred>`:** C types can contain private fields, platform-specific sizes, or alignment constraints that Rust cannot represent directly. `Opaque` tells Rust that the memory contains a C struct and must not be constructed, moved, or copied directly by Rust.
- **`#[repr(transparent)]`:** Guarantees that `Credential` has the exact same memory layout and ABI as the underlying raw C `struct cred`. This allows casting pointers between C and Rust safely.

---

## 2. Implementing Thread Safety

Because Rust cannot verify the thread safety of raw C structs, we must manually implement `Send` and `Sync` if the type meets safety criteria:

```rust,ignore
// SAFETY:
// - `Credential::dec_ref` can be called from any thread.
// - It is okay to send ownership of `Credential` across thread boundaries.
unsafe impl Send for Credential {}

// SAFETY: It's OK to access `Credential` through shared references from other threads because
// we're either accessing properties that don't change or that are properly synchronised by C code.
unsafe impl Sync for Credential {}
```

- **`Send`:** Allows passing ownership of `Credential` between threads.
- **`Sync`:** Allows multiple threads to access `&Credential` concurrently.

---

## 3. Safe Encapsulation

```rust,ignore
impl Credential {
    /// Returns a raw pointer to the inner credential.
    #[inline]
    pub fn as_ptr(&self) -> *const bindings::cred {
        self.0.get()
    }

    /// Returns the effective UID of the given credential.
    #[inline]
    pub fn euid(&self) -> Kuid {
        // SAFETY: By the type invariant, we know that `self.0` is valid. Furthermore, the `euid`
        // field of a credential is never changed after initialization, so there is no potential
        // for data races.
        Kuid::from_raw(unsafe { (*self.0.get()).euid })
    }

    /// Get the id for this security context.
    #[inline]
    pub fn get_secid(&self) -> u32 {
        let mut secid = 0;
        // SAFETY: The invariants of this type ensures that the pointer is valid.
        unsafe { bindings::security_cred_getsecid(self.0.get(), &mut secid) };
        secid
    }
}
```

- **Reading Immutable Fields:** Accessing `euid` is safe without locks because C credentials are write-once (immutable after initialization).
- **FFI Encapsulation:** `get_secid` calls a raw C function but hides the pointer manipulation and output argument allocation from the caller.

---

## 4. Reference Counting

To ensure a `Credential` allocation is never freed while Rust holds a reference, we implement the `AlwaysRefCounted` trait:

```rust,ignore
// SAFETY: The type invariants guarantee that `Credential` is always ref-counted.
unsafe impl AlwaysRefCounted for Credential {
    #[inline]
    fn inc_ref(&self) {
        // SAFETY: The existence of a shared reference means that the refcount is nonzero.
        unsafe { bindings::get_cred(self.0.get()) };
    }

    #[inline]
    unsafe fn dec_ref(obj: core::ptr::NonNull<Credential>) {
        // SAFETY: The safety requirements guarantee that the refcount is nonzero. The cast is okay
        // because `Credential` has the same representation as `struct cred`.
        unsafe { bindings::put_cred(obj.cast().as_ptr()) };
    }
}
```

- Implementing `AlwaysRefCounted` integrates `Credential` with Rust's reference-counting pointer `ARef<T>`.
- **`inc_ref`:** Calls C's `get_cred` to increment the refcount.
- **`dec_ref`:** Calls C's `put_cred` to decrement the refcount, which frees the object if the count drops to 0.

<details>

- Discuss the `Opaque` type.
- Discuss the `SAFETY:` comments.
- Show its constructor in `rust/kernel/fs/file.rs`.

</details>
