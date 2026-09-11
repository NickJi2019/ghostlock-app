# Native exploit backend decoupling log

This is the chronological record for
[`DECOUPLING_PLAN.md`](DECOUPLING_PLAN.md). It records observations, decisions,
implementation events, and validation evidence. All actionable items, unresolved
problems, and completion checkboxes belong in the plan.

## Observations

### 2026-09-11 — Unchecked MCAST profile geometry detected

  The validator checks only the lock write. It does not bound the target/value or
  task writes. The lock expression can overflow in 32-bit unsigned arithmetic
  before comparison with the buffer size. Both MCAST implementations allocate a
  profile-sized stack VLA without an upper bound. The resolution is tracked in
  Plan Phase 1 and the detected-issues section.

### 2026-09-11 — Backend-selection coupling detected

  MCAST is selected from `kernel_major == 5` plus a nonzero MCAST offset. TCP
  Zerocopy is selected from `compact_waiter`. These rules conflate transport
  choice with kernel version and waiter representation. The resolution is tracked
  in Plan Phases 2 and 4.

### 2026-09-11 — TCP-specific payload coupling detected

  `prepare_skb_payload()` changes payload delta, chunk bias, fake-task offset, and
  credential-copy offset by calling `tcp_route_selected()`. The resolution is
  tracked in Plan Phase 4.

### 2026-09-11 — TCP-derived W3 semantics detected

  W3 treats TCP as exact-target and every other backend as target-plus-eight.
  The resolution is tracked in Plan Phase 4.

### 2026-09-11 — Interleaved MCAST recovery detected

  Disarm, scratch quarantine/repair, fast credential repair, retry changes, and
  resident mode are spread across `main.c`, `fops.c`, and `util.c`. The resolution
  is tracked in Plan Phases 3 and 4.

### 2026-09-11 — Implicit zero sentinels detected

  Several fields cannot distinguish an omitted value from a valid explicit zero.
  This affects fallback behavior, JSON comparison, and validation. The resolution
  is tracked in Plan Phases 5, 7, and 8.

### 2026-09-11 — Partial MCAST initialization detected

  Thread creation or setup can fail before `mr_ready` is set. The current stop
  function then skips cleanup, potentially leaving live threads or descriptors.
  The resolution is tracked in Plan Phase 3.

## Decisions

### 2026-09-11 — Use backend terminology

PSELECT, TCP Zerocopy, and MCAST are waiter-overwrite backends within a shared
GhostLock flow. Names and profile fields describe the backend rather than a
kernel version.

### 2026-09-11 — Separate version, layout, backend, and recovery

- `kernel_major` describes the target kernel.
- `waiter_layout` describes the kernel data structure.
- `waiter_backend` describes how controlled data overlaps the stale waiter.
- Recovery configuration describes backend/target-specific side-effect repair.

## Validation results

No post-refactor device validation has been performed yet.
