---
minutes: 10
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Using Credentials in `File::cred`

[`rust/kernel/cred.rs`](https://github.com/Darksonn/linux/blob/rfl-course-edu/rust/kernel/cred.rs) & [`rust/kernel/fs/file.rs`](https://github.com/Darksonn/linux/blob/rfl-course-edu/rust/kernel/fs/file.rs):

```rust,ignore
impl Credential {
    /// Creates a reference to a [`Credential`] from a valid pointer.
    ///
    /// # Safety
    ///
    /// The caller must ensure that `ptr` is valid and remains valid for the
    /// lifetime of the returned [`Credential`] reference.
    pub unsafe fn from_ptr<'a>(ptr: *const bindings::cred) -> &'a Credential {
        // SAFETY: The safety requirements guarantee the validity of the
        // dereference, while the `Credential` type being transparent makes
        // the cast ok.
        unsafe { &*ptr.cast() }
    }
}

impl File {
    /// Returns the credentials of the task that originally opened the file.
    pub fn cred(&self) -> &Credential {
        // SAFETY: It's okay to read the `f_cred` field without synchronization
        // because `f_cred` is never changed after initialization of the file.
        let ptr = unsafe { (*self.as_ptr()).f_cred };

        // SAFETY: The signature of this function ensures that the caller will
        // only access the returned credential while the file is still valid,
        // and the C side ensures that the credential stays valid at least as
        // long as the file.
        unsafe { Credential::from_ptr(ptr) }
    }
}
```

- **`unsafe fn` (`# Safety`):** Preconditions the caller must uphold.
- **`unsafe { ... }` (`// SAFETY:`):** Proof that those preconditions hold.

<details>

- Explain the two distinct uses of the `unsafe` keyword and how they fit together:
  1. **`unsafe fn` (`Credential::from_ptr`):** Because raw pointer `ptr` has no lifetime, the compiler cannot check how long the `struct cred` stays alive. Marking `from_ptr` as `unsafe fn` with a `/// # Safety` comment shifts that obligation to its caller. (All raw C bindings in `kernel::bindings` are also implicitly `unsafe fn`.)
  2. **`unsafe { ... }` inside a safe `pub fn` (`File::cred`):** `File::cred` is **completely safe** to call. Its two `// SAFETY:` comments prove why the C subsystem's invariants plus Rust's type system satisfy the preconditions:
     - Reading `f_cred` without locking is safe because the VFS never mutates `f_cred` after file creation.
     - Calling `Credential::from_ptr(ptr)` is sound because `fn(&'a self) -> &'a Credential` ties the returned reference's lifetime `'a` to `&'a File`, and the C `struct file` holds a reference to `f_cred` for its entire lifetime.
- **What to check when reviewing Rust abstractions:**
  - Does the safe public API (`fn(&'a self) -> &'a Credential`) make it impossible for safe callers to violate the C subsystem's invariants?
  - Does every `// SAFETY:` comment accurately prove the `# Safety` requirements of the operation inside?

</details>
