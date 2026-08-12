---
minutes: 15
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Direct Memory Access (DMA)

Direct Memory Access (DMA) allows hardware devices to read and write system RAM directly without CPU intervention.

To prevent cache coherency bugs (where the CPU reads stale cache line values while hardware writes new data directly to RAM), we allocate **coherent DMA buffers**.

```rust,ignore
use kernel::dma::{DmaMask, CoherentBox, Coherent};

fn init_dma(pdev: &pci::Device<Core<'_>>) -> Result {
    // 1. Set the DMA mask (e.g. telling the kernel we support 28-bit DMA addresses):
    let mask = DmaMask::try_new(28)?;
    unsafe { pdev.dma_set_mask_and_coherent(mask)? };

    // 2. Allocate CPU-owned Coherent Box:
    // Guarantees cache-coherent memory mapped to virtual memory.
    let mut cbox: CoherentBox<u32> = CoherentBox::zeroed(pdev.as_ref(), GFP_KERNEL)?;
    
    // CPU can safely read/write to the box:
    *cbox = 0xbeef_beef;

    // 3. Share with the device:
    // Conversion yields a read-only view of the DMA-capable buffer.
    let shared_coherent: Coherent<u32> = cbox.into();
    
    // Retrieve the bus address to write into the device's DMA address registers:
    let bus_address = shared_coherent.dma_handle();
    
    Ok(())
}
```

## Physical vs. Virtual vs. Bus Addresses

1. **Virtual Address (`*mut T`):** Used by the CPU to read/write memory.
2. **Physical Address (`phys_addr_t`):** The actual address of the RAM chip.
3. **Bus Address (`dma_addr_t` / `DmaAddress`):** The address translated by the IOMMU that the device puts on the PCIe bus.

Coherent buffers guarantee that CPU virtual address writes and Device bus address writes are immediately visible to each other without manual cache flushing.

<details>

- Discuss the transition from `CoherentBox` (CPU-exclusive, mutable) to `Coherent` (Device-shared, immutable to the CPU to avoid races).
- Discuss DMA masks: why 28-bit mask is required for QEMU edu (since the edu hardware only registers 28 bits for DMA destination addresses).

</details>
