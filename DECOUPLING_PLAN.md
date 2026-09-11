# Native exploit backend decoupling and validation plan

This document tracks the decoupling work started from the review of
[`discussion_r3985917556`](https://github.com/YuKongA/ghostlock-app/pull/124#discussion_r3985917556).
It is a living plan: implementation progress is updated here as work proceeds.

All work items and unresolved issues are tracked in this plan. Chronological
observations, decisions, and validation events are recorded separately in
[`DECOUPLING_LOG.md`](DECOUPLING_LOG.md), which contains no authoritative task
state.

## Status

- Branch: `decoupling`
- Baseline: `ab18490`
- Last updated: 2026-09-11
- Overall status: planning
- Checkbox convention:
  - `[ ]` not started or not verified
  - `[x]` implemented and verified

## Goals and constraints

- [ ] Fix every bounds and arithmetic issue involved in profile-driven native
  buffer construction, beginning with the multicast stack VLA reported in the
  review.
- [ ] Select exploit behavior through explicit profile capabilities instead of
  inferring it from `kernel_major`, device identity, or a zero-valued field.
- [ ] Model PSELECT, TCP Zerocopy, and MCAST as waiter-overwrite backends behind
  one shared exploit flow.
- [ ] Model waiter layout independently from the overwrite backend.
- [ ] Isolate backend-specific payload geometry, write semantics, lifecycle, and
  recovery behavior.
- [ ] Distinguish an absent optional profile field from a field explicitly set to
  zero.
- [ ] Keep Xperia-specific values entirely in its profile.
- [ ] Mark MCAST as currently verified on Xperia Android 5.15 without restricting
  the implementation itself to kernel 5.15.
- [ ] Preserve the currently verified Xperia buffer geometry and timing unless a
  separately documented change is required.
- [ ] Preserve existing 6.x behavior while moving its implicit defaults into
  explicit backend and layout selections.

## Phase 0: inventory and behavior baseline

- [x] Confirm the worktree is on `decoupling` at baseline `ab18490`.
- [x] Identify current backends:
  - PSELECT: `do_pselect_fake_lock_route()`
  - TCP Zerocopy: `do_tcp_fake_lock_route()`
  - MCAST: `do_kernel5_fake_lock_route()`
- [x] Identify the current Xperia MCAST geometry:
  - buffer size: `0x108`
  - waiter offset: `0x60`
  - task offset: `0x30`
  - lock offset: `0x38`
- [x] Confirm that the current Xperia ranges fit in its buffer:
  - family: `[0x08, 0x0a)`
  - target/value: `[0x60, 0x70)`
  - task: `[0x90, 0x98)`
  - lock: `[0x98, 0xa0)`
- [ ] Record a concise call graph for W1, W2, W3, resident mode, and cleanup.
- [ ] Record every read of backend-, layout-, and recovery-related profile fields.
- [ ] Capture current build/test results before implementation.

## Phase 1: multicast memory-safety fix

- [ ] Define an explicit maximum MCAST option-buffer size consistent with the
  reclaim-page design.
- [ ] Remove both profile-sized stack VLAs from `src/core/fops.c`.
- [ ] Add overflow-safe helpers:
  - [ ] `range_fits(container_size, base, relative, width)`
  - [ ] checked `size_t` addition
  - [ ] checked `size_t` multiplication
  - [ ] checked `uintptr_t` addition
- [ ] Validate the fixed family write at `[8, 10)`.
- [ ] Validate the resident target/value write at
  `[waiter_offset, waiter_offset + 16)`.
- [ ] Validate the task pointer write.
- [ ] Validate the lock pointer write.
- [ ] Validate conversion of the option-buffer length to `socklen_t`.
- [ ] Validate lock-slot geometry, including
  `(slot_count - 1) * slot_stride` overflow.
- [ ] Validate fake-BSS, fake-lock, and fake-task address arithmetic.
- [ ] Make MCAST stamping return a structured failure instead of continuing after
  invalid geometry or a failed `setsockopt()`.
- [ ] Verify that the Xperia profile produces exactly the same bytes and option
  length after the safety changes.

## Phase 2: explicit backend and waiter-layout model

- [ ] Add `enum waiter_backend`:
  - [ ] `WAITER_BACKEND_UNSPECIFIED`
  - [ ] `WAITER_BACKEND_PSELECT`
  - [ ] `WAITER_BACKEND_TCP_ZEROCOPY`
  - [ ] `WAITER_BACKEND_MCAST`
- [ ] Add `enum waiter_layout`:
  - [ ] `WAITER_LAYOUT_UNSPECIFIED`
  - [ ] `WAITER_LAYOUT_COMPACT`
  - [ ] `WAITER_LAYOUT_RT_WAITER_NODE`
- [ ] Stop using `kernel_major == 5` to select MCAST.
- [ ] Stop using `compact_waiter` to select TCP Zerocopy.
- [ ] Replace `compact_waiter` with the explicit waiter-layout enum.
- [ ] Configure existing profiles explicitly:
  - [ ] Xperia 5.15: MCAST + COMPACT
  - [ ] supported 6.1 profiles: TCP_ZEROCOPY + COMPACT
  - [ ] supported 6.6 profiles: PSELECT + RT_WAITER_NODE
  - [ ] supported 6.12 profiles: PSELECT + RT_WAITER_NODE
- [ ] Retain `kernel_major` only for kernel description, compatibility checks, and
  tooling diagnostics.

## Phase 3: backend interface and naming

- [ ] Introduce a shared backend interface or an equivalent static dispatch
  design covering validation, preparation, triggering, and cleanup.
- [ ] Rename backend entry points:
  - [ ] `do_pselect_fake_lock_route()` to `do_pselect_waiter_overwrite()`
  - [ ] `do_tcp_fake_lock_route()` to `do_tcp_zerocopy_waiter_overwrite()`
  - [ ] `do_kernel5_fake_lock_route()` to `do_mcast_waiter_overwrite()`
- [ ] Rename MCAST resident functions:
  - [ ] `kernel5_resident_start()` to `mcast_resident_writer_start()`
  - [ ] `kernel5_resident_write()` to `mcast_resident_writer_write()`
  - [ ] `kernel5_resident_stop()` to `mcast_resident_writer_stop()`
- [ ] Rename internal `mr_*` state to MCAST-specific names.
- [ ] Replace numeric route-step ranges with structured backend results.
- [ ] Move backend-owned file descriptors, threads, mappings, and atomics into
  backend context structures where practical.
- [ ] Add comments stating that MCAST is currently verified on Xperia Android
  5.15 but selected through profile data and not inherently version-bound.
- [ ] Rename `GHOSTLOCK_5X_RESIDENT` and `GHOSTLOCK_5X_PHASE1_PROBE` to
  backend-based names.
- [ ] Decide and document whether legacy environment-variable aliases are retained
  temporarily.

## Phase 4: remove hidden payload and control-flow coupling

- [ ] Remove `tcp_route_selected()` decisions from `prepare_skb_payload()`.
- [ ] Move these payload properties into backend configuration or preparation:
  - [ ] payload base delta
  - [ ] chunk bias
  - [ ] fake-task offset
  - [ ] credential-copy offset
- [ ] Move compact/extended waiter construction out of PSELECT transport logic.
- [ ] Represent write behavior explicitly, including exact-target writes and
  rb-erase side writes.
- [ ] Stop using `tcp_route_selected()` to determine W3 target direction.
- [ ] Stop using `compact_waiter` directly to choose consumer nice behavior.
- [ ] Separate generic waiter lifecycle from backend overwrite triggering.
- [ ] Move MCAST disarm into the MCAST lifecycle.
- [ ] Move MCAST scratch repair behind an explicit recovery strategy.
- [ ] Move MCAST fast credential repair behind an explicit recovery strategy.
- [ ] Keep recovery selection in profile data rather than kernel-version checks.
- [ ] Audit W1/W2 retry counts and timing for remaining backend/version inference.

## Phase 5: profile structure and optional fields

- [ ] Group MCAST geometry into a dedicated configuration structure.
- [ ] Group credential-copy fields into a dedicated configuration structure.
- [ ] Replace four parallel credential-reference fields with a bounded array of
  `(offset, image)` entries.
- [ ] Add explicit presence tracking for optional values.
- [ ] Define optional/default semantics for:
  - [ ] `kernel_phys_load`
  - [ ] `pselect_waiter_shift`
  - [ ] `mm_struct_sz`
  - [ ] `kernelsnitch_collisions`
  - [ ] `requires_shizuku`
  - [ ] credential references
  - [ ] recovery strategy and parameters
- [ ] Document which fields are always required.
- [ ] Document which fields are required only by a selected backend.
- [ ] Document which fields are required only by a selected waiter layout.
- [ ] Document which fields are required only by a selected recovery strategy.
- [ ] Ensure a valid explicit zero is never interpreted as “absent” or “use
  fallback.”

## Phase 6: layered native validation

- [ ] Split `validate_offsets_profile()` into focused validators:
  - [ ] common profile validation
  - [ ] waiter-layout validation
  - [ ] backend validation
  - [ ] credential validation
  - [ ] recovery-strategy validation
- [ ] Require only fields consumed by the selected behavior.
- [ ] Produce field-specific validation errors with the invalid value and expected
  range.
- [ ] Audit all credential arithmetic:
  - [ ] usage-field range
  - [ ] capability count multiplication
  - [ ] capability range
  - [ ] reference count before array access
  - [ ] each reference range
  - [ ] destination offset plus copy size
- [ ] Audit kernel-address arithmetic and reject wraparound.
- [ ] Reject unsupported enum values and invalid combinations.
- [ ] Ensure compiled profiles and imported profiles pass through equivalent
  validation.
- [ ] Ensure failed validation leaves no partially published `active_offsets`.

## Phase 7: native JSON import hardening

- [ ] Make integer parsing detect decimal and hexadecimal overflow.
- [ ] Reject narrowing overflow for `uint8_t`, `uint32_t`, and `int32_t` fields.
- [ ] Preserve valid unsigned 64-bit ARM64 addresses.
- [ ] Treat a present but malformed field as an error rather than as absent.
- [ ] Detect duplicate fields that could make profile meaning ambiguous.
- [ ] Reject truncated files at the fixed input-buffer limit.
- [ ] Require valid delimiters after parsed numbers.
- [ ] Reject trailing non-whitespace content.
- [ ] Correctly skip mixed nested arrays and objects.
- [ ] Populate optional-field presence information during parsing.
- [ ] Define and test compatibility behavior for old JSON profiles.

## Phase 8: tooling, Android, and documentation synchronization

- [ ] Update the Rust extractor to emit backend and waiter-layout fields.
- [ ] Update generated C profile output.
- [ ] Update generated JSON profile output.
- [ ] Update Android imported-profile comparison for explicit zero and presence.
- [ ] Update supported-kernel generation for backend capabilities.
- [ ] Update English and Chinese README descriptions.
- [ ] Describe MCAST as currently verified on the Xperia 5.15 target.
- [ ] Avoid claiming that a backend is available for an entire kernel family
  without profile data and testing.
- [ ] Document profile migration and compatibility behavior.

## Phase 9: tests and verification

- [ ] Add focused native validation tests for:
  - [ ] current Xperia profile success
  - [ ] MCAST target/value overflow
  - [ ] MCAST task overflow
  - [ ] MCAST lock overflow
  - [ ] MCAST oversized buffer
  - [ ] lock-slot multiplication overflow
  - [ ] credential offset overflow
  - [ ] credential count multiplication overflow
  - [ ] missing versus explicit-zero optional fields
  - [ ] invalid enum combinations
  - [ ] JSON integer narrowing and overflow
- [ ] Verify every existing compiled profile.
- [ ] Verify old imported-profile compatibility according to the documented rule.
- [ ] Run `cargo test --manifest-path tools/extract_rs/Cargo.toml`.
- [ ] Build the native debug and release artifacts.
- [ ] Run `./gradlew --no-daemon --no-configuration-cache :app:compileDebugKotlin`.
- [ ] Build the Android debug and release APKs.
- [ ] Run `git diff --check`.
- [ ] Compare Xperia MCAST payload bytes against the baseline.
- [ ] Perform Xperia W1/W2 regression testing only after static and build checks
  pass.
- [ ] Record real-device results, including failures and timing, in this document.

## Suggested commit sequence

- [ ] Commit 1: harden MCAST geometry and remove profile-sized VLAs.
- [ ] Commit 2: add explicit backend/layout fields and migrate existing profiles.
- [ ] Commit 3: introduce backend dispatch and rename route functions.
- [ ] Commit 4: isolate payload geometry, write semantics, and recovery behavior.
- [ ] Commit 5: add optional-field presence and layered validation.
- [ ] Commit 6: harden JSON import and synchronize tooling/Android code.
- [ ] Commit 7: complete documentation and validation coverage.

## Detected issues requiring resolution

- [ ] **Unchecked MCAST profile geometry**  
  Bound the target/value, task, lock, and family writes; prevent arithmetic
  overflow; remove both profile-sized stack VLAs. Assigned to Phase 1.
- [ ] **Backend selection inferred from unrelated properties**  
  Replace `kernel_major == 5` and `compact_waiter` routing inference with explicit
  backend and waiter-layout fields. Assigned to Phases 2 and 4.
- [ ] **Shared payload builder knows the TCP backend**  
  Remove `tcp_route_selected()` from payload geometry decisions. Assigned to
  Phase 4.
- [ ] **W3 behavior inferred from the TCP backend**  
  Represent exact-target and target-plus-eight behavior as explicit write
  semantics. Assigned to Phase 4.
- [ ] **MCAST recovery interleaved with the generic exploit flow**  
  Isolate disarm, scratch repair, credential repair, retry behavior, and resident
  lifecycle. Assigned to Phases 3 and 4.
- [ ] **Optional values use zero as an implicit sentinel**  
  Add explicit field presence and preserve valid explicit zero values. Assigned
  to Phases 5, 7, and 8.
- [ ] **Resident MCAST startup has partial-initialization states**  
  Track every acquired resource and clean up correctly after partial startup.
  Assigned to Phase 3.

Add every newly detected problem to this section as an unchecked checkbox and
assign it to at least one implementation phase before continuing dependent work.

## Tracking log

- [x] Create a separate log for newly discovered issues, decisions, and validation
  results: [`DECOUPLING_LOG.md`](DECOUPLING_LOG.md).
- [ ] Keep all task state and unresolved-problem checkboxes in this plan.
- [ ] Keep the log limited to chronological records, decisions, and validation
  evidence.
