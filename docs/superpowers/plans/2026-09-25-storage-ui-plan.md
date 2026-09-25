# Storage, Product UI and Developer Tooling Implementation Plan

> **For agentic workers:** Execute after write/service gate.

**Goal:** Add durable history/backups/traces, full Compose product surfaces, declarative feature controls and engineering inspection tools.

**Architecture:** UI consumes application/use-case state only. Room stores normalized product history. Developer screens expose trace/evidence/variant explanations without bypassing runtime safety.

**Tech Stack:** Jetpack Compose, Navigation Compose, ViewModel, Coroutines/Flow, Room, Hilt, Android resource localization, Compose UI tests.

**Spec:** `docs/superpowers/specs/2026-09-25-autodiag-android-master-design.md`

## Global Constraints
- UI never sends raw diagnostic bytes.
- UI write actions route only through application use cases and the safety engine.
- English and Ukrainian resources are mandatory.
- Unsupported/protected/ambiguous states must be explicit.
- Backups and write transaction records are immutable.


## Review Focus
- database migration from old schema;
- very large trace session;
- unsupported feature state;
- accessibility with large text;
- process recreation during an active read-only screen.

---

### Task 1: Room schema

**Files:**
- `storage/.../AutoDiagDatabase.kt`
- Entities for vehicle, ECU, scan, DTC, live-data session/sample, backup, write transaction, operation history, trace, runtime pack, connection profile.
- DAO tests and migration tests.

- [ ] Write entity/DAO tests and migration baseline.
- [ ] Implement normalized schema with immutable backup/transaction records.
- [ ] Add migration-test harness.
- [ ] Commit `feat(storage): add AutoDiag Room persistence`.

### Task 2: Application use cases and state holders

**Files:**
- `core-runtime/.../usecase/ConnectVehicle.kt`
- `.../RunAutoScan.kt`
- `.../ObserveLiveData.kt`
- `.../ExecuteFeature.kt`
- `.../RunServiceTool.kt`

- [ ] Test use cases against fakes without Android UI.
- [ ] Implement state/result flows.
- [ ] Commit `feat(runtime): add application use cases`.

### Task 3: Compose navigation and top-level screens

**Files:**
- `ui/.../AutoDiagNavGraph.kt`
- Screens: Garage, Vehicle Overview, Health/Scan, ECUs, Faults, Live Data, Customizations, Service, History, Connection, Settings, Developer/Traces.

- [ ] Add Compose tests for navigation and state restoration.
- [ ] Implement independent UI design with EN/UK resources.
- [ ] Commit `feat(ui): add AutoDiag navigation and main product surfaces`.

### Task 4: Declarative feature renderer

**Files:**
- `ui/.../feature/FeatureRenderer.kt`
- controls for toggle, enum, range, numeric, action, wizard, service routine, read-only, protected/unsupported.

- [ ] Add tests for every interaction type and unavailable state.
- [ ] Render current/requested values and evidence/runtime availability.
- [ ] Route writes only through use case -> safety engine.
- [ ] Commit `feat(ui): add declarative feature controls`.

### Task 5: Write/service safety UX

**Files:**
- `ui/.../safety/WriteConfirmationScreen.kt`
- `.../WriteProgressScreen.kt`
- `.../RecoveryScreen.kt`

- [ ] Test target ECU/current value/requested value/prerequisite/backup display.
- [ ] Show success only after runtime success criterion/read-back verification.
- [ ] Preserve recovery view for interrupted/mismatch transactions.
- [ ] Commit `feat(ui): add write transaction safety UX`.

### Task 6: Developer tools

**Files:**
- `ui/.../developer/TraceViewer.kt`
- `.../VariantResolutionView.kt`
- `.../RuntimePackInspector.kt`
- `.../EcuIdentityView.kt`

- [ ] Add read-only engineering views for traces, evidence refs, operation plan and variant-selection explanation.
- [ ] Do not expose unrestricted production raw-write console.
- [ ] Commit `feat(devtools): add engineering inspection tools`.

### Gate
All product surfaces are functional against fake/replay data; write UI cannot bypass application/safety layers; accessibility checks pass for critical screens.
