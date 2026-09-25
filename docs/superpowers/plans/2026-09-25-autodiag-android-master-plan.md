# AutoDiag Android Master Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: execute the subsystem plans in the order listed below. Each subsystem plan is independently reviewable and must pass its own gate before the next begins.

**Goal:** Build the complete AutoDiag Android product described in the approved master specification, with data-driven vehicle/ECU variants, evidence-gated writes, diagnostics, live data, coding, adaptation, service tools, safety transactions, storage, UI, coverage tooling, and release validation.

**Architecture:** A multi-module Kotlin/Android application separates transport, adapter, protocol, OEM, feature, service, safety, storage, and UI concerns. Runtime feature packs are compiled from the external evidence repository and are never executed directly from research JSON.

**Tech Stack:** Kotlin, Jetpack Compose, Coroutines/Flow, Hilt, Room, kotlinx.serialization, Android Bluetooth APIs, Android USB Host, JUnit, Turbine, ktlint, detekt, GitHub Actions.

**Spec:** `docs/superpowers/specs/2026-09-25-autodiag-android-master-design.md`

## Global Constraints

- Namespace: `com.guns96x.autodiag`.
- Minimum Android API: 26.
- JDK toolchain: 17.
- No dynamic dependency versions.
- No direct UI-to-raw-protocol-byte calls.
- No executable write operation without EXACT evidence and deterministic applicability.
- No `exact_v[0]` or array-order variant selection.
- No research JSON consumed directly at runtime.
- No SFD/auth bypass, secret extraction, subscription/account emulation, or protected backend impersonation.
- Every write goes through the central safety transaction engine.
- Every material phase ends with tests, generated coverage, commit, and push.

## Review Focus

1. Ambiguous multi-variant features must block rather than choose a variant by order.
2. Adapter disconnect during a write must stop execution, preserve backup/trace, and never silently retry.
3. Runtime-pack corruption or incompatible schema must fail closed without damaging the last valid pack.
4. Protocol mismatch between ECU identity, feature variant and request builder must make the variant non-executable.
5. PARTIAL/UNRESOLVED/CONFLICT/PROTECTED evidence must never enter an executable write pack.

## Execution Order

1. `2026-09-25-foundation-evidence-plan.md`
2. `2026-09-25-transports-adapters-plan.md`
3. `2026-09-25-protocols-plan.md`
4. `2026-09-25-vehicle-diagnostics-livedata-plan.md`
5. `2026-09-25-feature-writing-service-plan.md`
6. `2026-09-25-storage-ui-plan.md`
7. `2026-09-25-import-coverage-release-plan.md`

## Phase Gates

### Gate A — Foundation
Required before transport/protocol implementation:
- multi-module Gradle build passes;
- canonical domain model compiles;
- runtime-pack schema exists;
- evidence compiler rejects known-invalid fixtures;
- CI runs unit/lint/schema jobs.

### Gate B — Communications
Required before vehicle detection:
- SPP/BLE/USB transports expose the same `Transport` interface;
- adapter capability probing works against fakes;
- protocol-independent byte traces are recorded;
- reconnect and cancellation tests pass.

### Gate C — Protocols
Required before diagnostics:
- CAN, ISO-TP, UDS, KWP2000, TP2.0 golden vectors pass;
- protocol sessions expose normalized request/response APIs;
- timeout/NRC/response-pending paths are covered;
- no OEM logic exists in protocol modules.

### Gate D — Read-only Product
Required before write features:
- adapter -> vehicle -> ECU discovery -> identity -> AutoScan -> DTC -> live data works in trace replay;
- applicability can resolve read-only variants;
- no write UI is enabled yet.

### Gate E — Writes and Services
Required before full UI/release:
- coding/adaptation/service plans are data-driven;
- central safety engine controls all writes;
- backup/read-back/interruption tests pass;
- ambiguous or protected variants are blocked.

### Gate F — Coverage Closure
Required for release:
- full evidence import performed;
- all catalog features classified;
- all executable variants have tests;
- coverage reports generated from data;
- PQ35 Golf 5 acceptance baseline executed;
- unresolved/protected report included with release.

## Branch/Commit Discipline

For every task:
1. add failing test;
2. run the focused test and confirm failure;
3. implement the smallest correct change;
4. run focused tests;
5. run module tests;
6. run relevant validation tools;
7. commit with one logical change;
8. push;
9. continue only after the phase remains green.

## Final Deliverables

- release APK;
- runtime evidence pack;
- feature/variant/protocol/platform coverage reports;
- golden vector suite;
- trace replay suite;
- write-safety suite;
- Room migration suite;
- PQ35 real-vehicle acceptance report;
- blocked/protected evidence report;
- architecture and protocol docs synchronized with code.
