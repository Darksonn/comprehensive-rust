---
minutes: 10
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# IO Polling

```rust,ignore
pub fn read_poll_timeout<Op, Cond, T>(
    mut op: Op,
    mut cond: Cond,
    sleep_delta: Delta,
    timeout_delta: Delta,
) -> Result<T>
where
    Op: FnMut() -> Result<T>,
    Cond: FnMut(&T) -> bool,
```

```rust,ignore
use kernel::{
    io::{poll::read_poll_timeout, register, Io},
    pci,
    time::Delta,
};

register! {
    STATUS(u32) @ 0x20 {
        0:0 busy;
    }
}

fn wait_until_idle(bar: &pci::Bar<'_, 0x80>) -> Result {
    read_poll_timeout(
        || Ok(bar.read(STATUS)),
        |status: &STATUS| status.busy().get() == 0,
        Delta::from_millis(1),
        Delta::from_millis(100),
    )?;
    Ok(())
}
```

<details>

- **Polling vs. Interrupts:** When a hardware operation takes time to complete, a driver can either wait for an interrupt or poll a status register.
- **Sleeping poll (`read_poll_timeout`):** Used in process context (such as an `ioctl` handler).
  - `op`: Closure that reads the register and returns `Result<T>`. Because `bar.read(STATUS)` is compile-time bounds-checked and infallible, we wrap it in `Ok(...)` (or use `bar.try_read(STATUS)`).
  - `cond`: Closure returning `true` when the target state is reached.
  - `sleep_delta`: How long to sleep between reads.
  - `timeout_delta`: Maximum total wait time before returning `Err(ETIMEDOUT)`.
- **Atomic context (`read_poll_timeout_atomic`):** If polling inside an interrupt handler or while holding a `SpinLock`, use `read_poll_timeout_atomic`, which busy-waits (`udelay`) instead of sleeping.

</details>
