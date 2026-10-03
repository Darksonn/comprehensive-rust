---
minutes: 10
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Allocation Failure

```rust,ignore
use kernel::prelude::*;

fn my_numbers() -> Result<KVec<i32>> {
    let mut numbers = KVec::new();

    // push() returns Result<(), AllocError> and must be handled with `?`:
    numbers.push(72, GFP_KERNEL)?;
    numbers.push(108, GFP_KERNEL)?;
    numbers.push(200, GFP_KERNEL)?;

    Ok(numbers)
}
```

<details>

- Contrast user-space OOM handling (abort/panic) with kernel-space requirement to return `-ENOMEM`.
- Try removing the use `?` and see the compilation error.
- Mention `KVec::reserve` for pre-allocating capacity before critical sections.

</details>
