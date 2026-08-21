---
minutes: 10
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# IO Polling

When interacting with hardware, some operations (such as starting a computation or resetting a device) take time to complete. The driver needs a way to wait until the hardware is ready.

There are two primary ways to do this:
1.  **Interrupts:** The hardware signals the CPU when it is done. This is the most efficient method but requires complex state synchronization.
2.  **Polling:** The CPU repeatedly reads a status register until it indicates completion.

In a process context (such as handling an IOCTL), we can sleep between poll attempts to avoid busy-waiting and yielding CPU cycles to other tasks.

---

## The `read_poll_timeout` Function

The Rust kernel abstractions provide **`kernel::io::poll::read_poll_timeout`** to safely implement sleeping poll loops.

It has the following signature:

```rust
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

### Parameters:
*   **`op`**: A closure that performs the read operation. It must return a `Result<T>`.
*   **`cond`**: A closure that takes the value `&T` returned by `op` and returns `true` if the condition is met (e.g., hardware is idle).
*   **`sleep_delta`**: The duration to sleep (`Delta`) between poll attempts. If `0`, it does not sleep (but still relaxes the CPU).
*   **`timeout_delta`**: The maximum duration to poll before returning an error.

---

## Example Usage

For the QEMU `edu` device, we can poll the `STATUS` register to wait until the factorial computation is finished (i.e. the `computing` bit becomes `0`):

```rust,ignore
use kernel::io::poll;
use kernel::time;

fn wait_for_factorial(bar: &pci::Bar<'_, { regs::END }>) -> Result {
    poll::read_poll_timeout(
        // 1. Read the STATUS register:
        || Ok(bar.read(regs::STATUS)),
        // 2. Wait until the 'computing' bit is 0 (idle):
        |status: &regs::STATUS| status.computing().get() == 0,
        // 3. Sleep 1 millisecond between polls:
        time::Delta::from_millis(1),
        // 4. Time out after 100 milliseconds:
        time::Delta::from_millis(100),
    )?;

    Ok(())
}
```

### Error Handling

*   If `op` returns an error (e.g., bus error), `read_poll_timeout` aborts and returns that error immediately.
*   If `timeout_delta` is exceeded before `cond` becomes `true`, it returns `Err(ETIMEDOUT)`.

---

## Atomic Polling

If you need to poll in an atomic context (such as an interrupt handler or while holding a spinlock) where sleeping is not allowed, you must use **`read_poll_timeout_atomic`** instead. This function performs a busy-wait (using `udelay`) and relaxes the CPU without sleeping.
