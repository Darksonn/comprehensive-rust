---
minutes: 15
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Interrupt Handling

Devices signal asynchronous hardware events (such as DMA transfer completion) via hardware interrupts. In Rust, we handle these by implementing the **`irq::Handler`** trait.

With the new lifetime-bound IRQ subsystem, the handler itself can be parameterised by the device lifetime. This allows it to borrow hardware resources (like MMIO BARs) directly, eliminating the need for `unsafe` data lookups inside the ISR.

```rust,ignore
use kernel::irq;
use kernel::device;

struct EduIrqHandler<'bound> {
    pdev: &'bound pci::Device<Bound>,
    bar: pci::Bar<'bound, { regs::END }>,
}

// 1. Implement the irq::Handler trait:
impl<'bound> irq::Handler for EduIrqHandler<'bound> {
    fn handle(&self) -> irq::IrqReturn {
        // ISR code runs in interrupt context!
        // We must check if the interrupt was ours, clear the hardware interrupt status,
        // and return.
        let status = *self.bar.read(regs::IRQ_STATUS).val();
        if status == 0 {
            return irq::IrqReturn::None;
        }

        dev_info!(self.pdev, "Interrupt handled! status=0x{:x}\n", status);
        self.bar.write_reg(regs::IRQ_STATUS::zeroed().with_val(status));
        
        irq::IrqReturn::Handled
    }
}

// 2. Request and register the IRQ in probe():
fn register_irq<'bound>(
    pdev: &'bound pci::Device<Core<'_>>,
    bar: pci::Bar<'bound, { regs::END }>,
) -> Result<irq::Registration<'bound, EduIrqHandler<'bound>>> {
    // Allocate 1 MSI/MSI-X vector:
    let vectors = pdev.alloc_irq_vectors(1, 1, pci::IrqTypes::all())?;
    let vector = *vectors.start();

    // SAFETY: We promise to store the returned Registration in a struct
    // (like EduDriverData or DRM RegistrationData) that is dropped on unbind,
    // ensuring it is never leaked/forgotten.
    let irq_init = unsafe {
        pdev.request_irq(
            vector,
            irq::Flags::SHARED,
            c"qemu_edu",
            try_pin_init!(EduIrqHandler {
                pdev: &**pdev,
                bar,
            }),
        )?
    };
    
    // In-place initialization is required because Registration is pinned
    // and holds the handler.
    Ok(irq_init)
}
```

## Interrupt Context Rules (The ISR)

The `handle` function runs in **interrupt context (atomic context)**:
- **No Blocking/Sleeping:** You cannot acquire a `Mutex`, allocate memory with `GFP_KERNEL`, or perform blocking I/O inside the handler.
- **Spinlocks:** If you need to lock shared state in an ISR, you must use `SpinLock` (never `Mutex`).
- **Memory Safety:** When `irq::Registration` is dropped, it automatically deregisters the handler (calling `free_irq`) and blocks until any running ISR completes. This guarantees that the handler (and the borrowed `bar`) cannot be dropped while the ISR is executing.

<details>

- Discuss how `alloc_irq_vectors` configures MSI-X/MSI or legacy IRQs.
- Explain `IrqReturn::Handled` (our device triggered it) vs `IrqReturn::None` (a shared IRQ line where another device triggered it).
- Emphasize why `request_irq` is `unsafe`: leaking the returned `Registration` (e.g. via `mem::forget`) would result in the kernel keeping a dangling pointer to the handler (and the borrowed BAR) in its interrupt table after the driver is unbound, causing a crash on the next interrupt.

</details>
