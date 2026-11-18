---
minutes: 15
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Interrupts

```rust,ignore
#[pin_data]
struct TestDrmData<'drm> {
    #[pin]
    _irq: irq::Registration<'vectors, TestIrqHandler<'bar, 'drm>>,
    vectors: pci::IrqVectorRegistration<'drm>,
    bar: pci::Bar<'drm, { regs::END }>,
}

#[pin_data]
struct TestIrqHandler<'bar, 'bound> {
    pdev: &'bound pci::Device<Bound>,
    bar: &'bar pci::Bar<'bound, { regs::END }>,
}

impl irq::Handler for TestIrqHandler<'_, '_> {
    fn handle(&self) -> irq::IrqReturn {
        dev_info!(self.pdev, "IRQ handled!\n");
        irq::IrqReturn::Handled
    }
}
```

In `pci::Driver::probe`:

```rust,ignore
let reg_data = try_pin_init!(TestDrmData {
    bar,
    vectors: pdev.alloc_irq_vectors(1, 1, pci::IrqTypes::all())?,
    // SAFETY: `_irq` is stored in `TestDrmData` and dropped before PCI unbind.
    _irq <- unsafe {
        irq::Registration::new(
            vectors.index(0)?.into(),
            irq::Flags::SHARED,
            c"pci_testdev_drm",
            try_pin_init!(TestIrqHandler { pdev, bar }),
        )
    },
});
```

<details>

- **Self-referential `#[pin_data]` lifetimes (`'vectors`, `'bar`):**
  - `_irq` borrows `vectors` (via `'vectors`) and `bar` (via `'bar`) inside the same pinned `TestDrmData` struct.
  - Because Rust drops struct fields in declaration order, `_irq` is declared **before** `vectors` and `bar` so the IRQ handler is unregistered (`free_irq`) before the vectors are freed or the BAR is unmapped.
  - Inside `try_pin_init!`, fields can be initialized in dependency order (`bar` and `vectors` first, then `_irq` borrowing them).
- **Interrupt Context Rules (`handle(&self)`):**
  - Runs in **atomic / hardirq context**: never sleep, never lock a `Mutex`, never allocate with `GFP_KERNEL`. Use `SpinLock` or atomics if you need to mutate shared state.
  - Return `IrqReturn::Handled` if our device raised the interrupt, or `IrqReturn::None` if the interrupt line is shared and our device's status register is `0`.
- **Why `irq::Registration::new` is `unsafe`:** If the caller leaked (`mem::forget`) the `irq::Registration` instead of dropping it before PCI `unbind`, the kernel would keep a dangling pointer to `TestIrqHandler` past `'bound`. When dropped normally, `irq::Registration` calls `free_irq()`, which waits for any running ISR to finish before returning.

</details>
