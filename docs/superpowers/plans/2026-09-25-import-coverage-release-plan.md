# Full Evidence Import, Coverage Closure, Validation and Release Plan

> **For agentic workers:** Execute after all runtime/UI subsystem gates.

**Goal:** Import the complete eligible evidence set, classify every feature/variant, close implementation gaps, validate on real hardware, and produce release artifacts with generated coverage reports.

**Architecture:** Coverage is generated from source evidence + runtime implementation + tests. Missing evidence lowers status; it never produces guessed code.

**Spec:** `docs/superpowers/specs/2026-09-25-autodiag-android-master-design.md`

## Review Focus
- source evidence commit changes under same feature ID;
- conflicting variant applicability;
- feature implemented but no trace/golden test;
- runtime pack rollback after bad update;
- real vehicle differs from replay fixture.

---

### Task 1: Import complete ghidracarista evidence

**Files:**
- Tool configuration under `tools/evidence-pack-compiler`
- Generated pack under `runtime-packs/vag/`
- Generated rejection report under `build/reports/evidence/`

- [ ] Pin source repository commit and artifact hashes.
- [ ] Run schema/provenance/variant/protocol validators.
- [ ] Compile complete VAG pack.
- [ ] Reject every non-reproducible/conflicting entry from executable states.
- [ ] Commit source configuration and generated manifest, not transient build logs.
- [ ] Commit `data(vag): import validated VAG evidence runtime pack`.

### Task 2: Generate coverage reports

**Files generated:**
- `coverage-summary.json`
- `coverage-summary.md`
- `feature-coverage.csv`
- `variant-coverage.csv`
- `protocol-coverage.md`
- `platform-coverage.md`
- `blocked-evidence.md`

- [ ] Generate reports from data.
- [ ] Assert catalog coverage accounting sums from actual records, not constants.
- [ ] Ensure every blocked feature names the exact missing/conflicting evidence class.
- [ ] Commit `docs(coverage): publish generated VAG coverage reports`.

### Task 3: Close runtime implementation gaps

- [ ] For every `WRITE_READY` or `READ_READY` entry without `RUNTIME_IMPLEMENTED`, add the missing supported generic codec/operation-plan type or mark a justified blocker.
- [ ] Add focused golden vectors for each newly executable operation shape.
- [ ] Re-run coverage until no evidence-ready entry lacks a runtime implementation.
- [ ] Commit in small operation-family commits.

### Task 4: Full trace-replay suite

**Files:**
- `testkit/src/testFixtures/traces/`
- `app/src/androidTest/.../ReplayAcceptanceTest.kt`

- [ ] Run PQ35, MQB and MEB evidence-backed replay fixtures where available.
- [ ] Inject disconnect, timeout, NRC, pending response, malformed frame and read-back mismatch.
- [ ] Assert transaction history and traces.
- [ ] Commit `test: complete multi-platform trace replay acceptance suite`.

### Task 5: PQ35 Golf 5 BLS real-vehicle acceptance

**Target:**
- VW Golf 5
- 1.9 TDI BLS
- PQ35
- Bosch EDC16U34

**Procedure:**
- [ ] Record adapter identity/capabilities.
- [ ] Run read-only discovery and save trace.
- [ ] Verify ECU identity results.
- [ ] Run AutoScan and DTC read.
- [ ] Run evidence-backed live-data reads.
- [ ] Verify coding/adaptation reads for confirmed features.
- [ ] Before any write, confirm backup/read-back tooling and choose only an evidence-complete low-risk test feature.
- [ ] Execute write through SafetyRuntime, read back, then restore original value if the exact reverse operation is supported.
- [ ] Save acceptance report and trace hashes.

### Task 6: Additional platform validation gates

- [ ] MQB write support remains below VEHICLE_VALIDATED until one real MQB acceptance vehicle passes.
- [ ] MEB write support remains below VEHICLE_VALIDATED until one real MEB acceptance vehicle passes.
- [ ] Static evidence/trace-only platforms use TRACE_VALIDATED or lower state.

### Task 7: Release hardening

**Files:**
- `docs/release/release-checklist.md`
- `docs/release/runtime-pack-recovery.md`
- `docs/testing/real-vehicle-procedure.md`

- [ ] Run `./gradlew clean check assembleRelease`.
- [ ] Run runtime-pack validation from clean checkout.
- [ ] Run Room migration tests.
- [ ] Verify EN/UK resources and accessibility checks.
- [ ] Verify app/runtime-pack version metadata.
- [ ] Produce signed release artifact according to repository release configuration.
- [ ] Publish final generated coverage and blocked/protected reports.
- [ ] Commit `release: prepare AutoDiag Android validated release`.

### Final Acceptance

A reviewer must be able to choose any executable feature and trace:

```
UI -> FeatureDefinition -> FeatureVariant -> applicability -> ECU identity
-> protocol -> OperationPlan -> request/response -> ValueCodec
-> WriteTransaction -> EvidenceRef -> tests
```

If that chain is incomplete, the feature is not release-accepted.
