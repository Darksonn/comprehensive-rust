---
minutes: 15
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Interrupt Handling

Devices signal asynchronous hardware events (such as DMA transfer completion) via hardware interrupts. In Rust, we handle these by implementing the **`irq::Handler`** trait.

```rust,ignore
use kernel::irq;
use kernel::device;

struct EduIrqHandler {
    pdev: ARef<pci::Device>,
}

// 1. Implement the irq::Handler trait:
impl irq::Handler for EduIrqHandler {
    fn handle(&self, _dev: &device::Device<Bound>) -> irq::IrqReturn {
        // ISR code runs in atomic/interrupt context!
        // We must check if the interrupt was ours, clear the hardware interrupt status,
        // and return.
        
        irq::IrqReturn::Handled
    }
}

// 2. Request and register the IRQ in probe():
fn register_irq(pdev: &pci::Device<Core<'_>>, handler: EduIrqHandler) -> Result<irq::Registration<EduIrqHandler>> {
    // Allocate 1 MSI/MSI-X vector:
    let vectors = pdev.alloc_irq_vectors(1, 1, pci::IrqTypes::all())?;
    let vector = *vectors.start();

    // Register our handler to the vector:
    pdev.request_irq(
        vector,
        irq::Flags::SHARED,
        c"qemu_edu",
        try_pin_init!(EduIrqHandler {
            pdev: pdev.into(),
        }),
    )
}
```

## Atomic Context Rules (The ISR)

The `handle` function runs in **interrupt context (atomic context)**:
- **No Blocking/Sleeping:** You cannot acquire a `Mutex`, allocate memory with `GFP_KERNEL`, or perform blocking I/O inside the handler.
- **Spinlocks:** If you need to lock shared state in an ISR, you must use `SpinLock` (never `Mutex`).
- **Drop-Safety:** When `irq::Registration` is dropped, it automatically deregisters the handler and blocks until any running ISR completes, preventing use-after-free bugs.

<details>

- Discuss how `alloc_irq_vectors` configures MSI-X/MSI or legacy IRQs.
- Explain `IrqReturn::Handled` (our device triggered it) vs `IrqReturn::None` (a shared IRQ line where another device triggered it).

</details>
