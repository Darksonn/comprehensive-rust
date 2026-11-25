---
minutes: 5
---

<!--
Copyright 2026 Google LLC
SPDX-License-Identifier: CC-BY-4.0
-->

# Condition Variables in C

The same `Channel` example written directly against the kernel's wait queue
API:

```c
struct channel {
    struct mutex lock;
    wait_queue_head_t wq;
    /* Protected by `lock`: */
    bool ready;
};

int wait_until_ready(struct channel *ch)
{
    DEFINE_WAIT(wait);

    mutex_lock(&ch->lock);
    while (!ch->ready) {
        prepare_to_wait_exclusive(&ch->wq, &wait, TASK_INTERRUPTIBLE);
        mutex_unlock(&ch->lock);
        schedule();
        finish_wait(&ch->wq, &wait);
        mutex_lock(&ch->lock);
        if (signal_pending(current)) {
            mutex_unlock(&ch->lock);
            return -ERESTARTSYS;
        }
    }
    mutex_unlock(&ch->lock);
    return 0;
}

void mark_ready(struct channel *ch)
{
    mutex_lock(&ch->lock);
    ch->ready = true;
    mutex_unlock(&ch->lock);
    wake_up_all(&ch->wq);
}
```

<details>

- **No link between lock and data:** Nothing in C ties `ready` or `wq` to
  `lock`. In Rust, `wait_interruptible(&mut guard)` demands proof that the lock
  is held, and `ready` can only be reached through the `Mutex`.
- **Easy-to-make mistakes:** In C it is easy to forget `mutex_unlock()` on the
  `-ERESTARTSYS` path, or to read `ch->ready` without holding the lock. Rust's
  `Guard` unlocks automatically on every return path and makes unlocked access
  impossible.
- **`wait_event_interruptible()`:** Many C drivers use the
  `wait_event_interruptible(wq, condition)` macro instead, which hides the
  prepare/schedule/finish loop shown here.
- **`wake_up_all()` is `notify_all()`:** Both call
  `__wake_up(wq, TASK_NORMAL, 0, NULL)`; `notify_one()` passes `1` instead.

</details>
