# Feature Runtime, Coding, Adaptation, Service and Safety Implementation Plan

> **For agentic workers:** Execute after the read-only product gate.

**Goal:** Implement deterministic variant selection, coding/adaptation/basic-settings/service execution and the central safety transaction engine.

**Architecture:** Feature runtime selects exactly one applicable EXACT variant. Operation plans are executed by specialized runtimes. All writes are mediated by SafetyRuntime and immutable backups.

**Spec:** `docs/superpowers/specs/2026-09-25-autodiag-android-master-design.md`

## Review Focus
- two exact variants match;
- ECU identity changes between read and write;
- read-back mismatch;
- service routine remains in progress;
- disconnect after write before verification.

---

### Task 1: Applicability and variant selector

**Files:**
- `feature-runtime/.../ApplicabilityEngine.kt`
- `.../VariantSelector.kt`
- `.../VariantSelectionResult.kt`

**Interface:**
```kotlin
suspend fun select(
    feature: FeatureDefinition,
    vehicle: VehicleIdentity,
    ecus: List<EcuIdentity>
): VariantSelectionResult
```

- [ ] Test EXACT_MATCH, NO_MATCH, AMBIGUOUS, PARTIAL_ONLY, PROTECTED, CONFLICT.
- [ ] Implement specificity comparison without array-order priority.
- [ ] If two exact variants remain equally applicable, return AMBIGUOUS.
- [ ] Commit `feat(feature): add deterministic applicability and variant selection`.

### Task 2: Coding runtime

**Files:**
- `coding-runtime/.../CodingRuntime.kt`
- `.../CodingBlock.kt`
- `.../CodingMutation.kt`

- [ ] Test long coding read/modify/write/read-back with unrelated-byte preservation.
- [ ] Test submodule coding and mismatch detection.
- [ ] Implement through operation plans only.
- [ ] Commit `feat(coding): add safe coding runtime`.

### Task 3: Adaptation runtime

**Files:**
- `adaptation-runtime/.../AdaptationRuntime.kt`
- `.../UdsAdaptationExecutor.kt`
- `.../Tp20AdaptationExecutor.kt`

- [ ] Add UDS DID adaptation fixture.
- [ ] Add TP2/KWP verified sequence fixture: select channel -> read -> modify -> write -> read-back.
- [ ] Explicitly model the evidenced 31 B9 / 31 BA / 31 BB application flow where applicable.
- [ ] Reject synthetic 0x21/0x27 mappings unless exact evidence pack supplies such a request builder.
- [ ] Commit `feat(adaptation): add UDS and TP2 adaptation runtimes`.

### Task 4: Basic settings and service runtime

**Files:**
- `basic-settings-runtime/.../BasicSettingsRuntime.kt`
- `service-runtime/.../ServiceRuntime.kt`
- `.../RoutinePoller.kt`

- [ ] Test prerequisite failure, start, in-progress polling, success, terminal failure, cleanup and cancellation.
- [ ] Add EPB and DPF operation-plan fixtures from verified evidence.
- [ ] Keep battery registration/service reset blocked until evidence pack contains complete executable plans.
- [ ] Commit `feat(service): add stateful basic-settings and service runtime`.

### Task 5: Central safety transaction engine

**Files:**
- `safety-runtime/.../WriteTransactionEngine.kt`
- `.../WriteTransactionState.kt`
- `.../WritePrecondition.kt`
- `.../RecoveryPolicy.kt`

- [ ] Test exact sequence: identity -> applicability -> capability -> voltage/preconditions -> read -> backup -> write -> positive response -> read-back -> compare.
- [ ] Test low voltage, disconnect, NRC, timeout, identity change, mismatch, unsupported adapter.
- [ ] Implement no automatic retry of non-idempotent writes.
- [ ] Commit `feat(safety): add evidence-gated write transaction engine`.

### Task 6: Backup and rollback policy

**Files:**
- `safety-runtime/.../BackupRecord.kt`
- `.../RollbackEvaluator.kt`

- [ ] Test immutable backup generation.
- [ ] Test rollback allowed only when reverse write is explicit and ECU remains reachable.
- [ ] Test rollback blocked path preserves recovery data.
- [ ] Commit `feat(safety): add immutable backups and explicit rollback policy`.

### Gate
No UI or runtime path can write directly without `WriteTransactionEngine`. Ambiguous, partial, conflict and protected variants are unexecutable.
