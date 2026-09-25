# Foundation and Evidence Pipeline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: execute task-by-task with tests before implementation.

**Goal:** Create the Android multi-module foundation, canonical domain model, evidence schema, runtime-pack compiler, validators, coverage skeleton, and CI gates.

**Architecture:** The Android app consumes only normalized runtime packs. Research evidence is imported by host-side tooling, validated, normalized into canonical models, and compiled into immutable versioned packs.

**Tech Stack:** Gradle Kotlin DSL, Kotlin/JVM tooling modules, kotlinx.serialization, JSON Schema, JUnit, Hilt baseline, GitHub Actions.

**Spec:** `docs/superpowers/specs/2026-09-25-autodiag-android-master-design.md`

## Global Constraints
- Namespace `com.guns96x.autodiag`.
- minSdk 26, JDK 17.
- No executable runtime pack entry without EXACT evidence.
- No hard-coded feature-count assertions.
- Runtime packs contain data only.

## Review Focus
- malformed runtime pack;
- duplicate variant IDs;
- ambiguous executable variants;
- evidence reference without source hash;
- schema version incompatibility.

---

### Task 1: Multi-module Gradle foundation

**Files:**
- Create: `settings.gradle.kts`
- Create: `build.gradle.kts`
- Create: `gradle/libs.versions.toml`
- Create: `app/build.gradle.kts`
- Create module build files for: `core-model`, `core-evidence`, `core-runtime`, `testkit`, `tools:evidence-pack-compiler`
- Create: `.github/workflows/ci.yml`

**Interfaces:**
- Produces stable module boundaries and pinned dependency aliases used by every later plan.

- [ ] **Step 1:** Create a Gradle test that runs `./gradlew projects` in CI and asserts all required modules are present.
- [ ] **Step 2:** Run `./gradlew projects`; expected failure because modules do not exist.
- [ ] **Step 3:** Create root/module Gradle files, Android namespace, minSdk 26, JDK 17 toolchain, Kotlin/Compose/Hilt/serialization/test dependencies through the version catalog.
- [ ] **Step 4:** Run `./gradlew projects assembleDebug`; expected PASS.
- [ ] **Step 5:** Add ktlint/detekt configuration and CI jobs for format check, detekt, unit tests, assembleDebug.
- [ ] **Step 6:** Run the same commands locally.
- [ ] **Step 7:** Commit `build: create AutoDiag multi-module foundation`.

### Task 2: Canonical domain model

**Files:**
- Create: `core-model/src/main/kotlin/com/guns96x/autodiag/model/VehicleIdentity.kt`
- Create: `.../EcuIdentity.kt`
- Create: `.../EcuVariant.kt`
- Create: `.../FeatureDefinition.kt`
- Create: `.../FeatureVariant.kt`
- Create: `.../OperationPlan.kt`
- Create: `.../ProtocolStep.kt`
- Create: `.../ValueCodec.kt`
- Create: `.../EvidenceRef.kt`
- Create: `.../EvidenceStatus.kt`
- Test: `core-model/src/test/.../ModelSerializationTest.kt`

**Interfaces:**
- Produces immutable serializable types consumed by evidence compiler, runtime, OEM, safety and UI modules.

- [ ] **Step 1:** Write serialization round-trip tests for one vehicle, one ECU, one feature with two variants, one operation plan and evidence refs.
- [ ] **Step 2:** Run module tests; expected FAIL.
- [ ] **Step 3:** Implement sealed/value types exactly matching the approved spec. Use explicit enums for evidence status and protocol-step type.
- [ ] **Step 4:** Add invariant constructors/helpers that reject blank IDs, negative byte offsets, zero masks for bitfield codecs, and executable variants without evidence.
- [ ] **Step 5:** Run tests; expected PASS.
- [ ] **Step 6:** Commit `feat(model): add canonical AutoDiag domain model`.

### Task 3: Runtime-pack schema and loader contract

**Files:**
- Create: `core-evidence/src/main/kotlin/.../RuntimePack.kt`
- Create: `.../RuntimePackManifest.kt`
- Create: `.../RuntimePackValidator.kt`
- Create: `core-evidence/src/main/resources/runtime-pack.schema.json`
- Test: `core-evidence/src/test/.../RuntimePackValidatorTest.kt`

**Interfaces:**
- Consumes canonical model.
- Produces `RuntimePackValidationResult` and validated `RuntimePack`.

- [ ] **Step 1:** Add fixtures for valid pack, corrupted hash, unsupported schema, executable PARTIAL variant, duplicate variant IDs, broken evidence ref.
- [ ] **Step 2:** Run tests; expected FAIL.
- [ ] **Step 3:** Implement manifest fields: schemaVersion, packVersion, buildCommit, sourceCommit, sourceArtifactHashes, generatedAt, featureCount, variantCount, evidenceSummary, payloadHash.
- [ ] **Step 4:** Implement fail-closed validator for all fixture cases.
- [ ] **Step 5:** Run tests; expected PASS.
- [ ] **Step 6:** Commit `feat(evidence): add runtime pack schema and fail-closed validation`.

### Task 4: Evidence importer and compiler

**Files:**
- Create: `tools/evidence-pack-compiler/src/main/kotlin/.../Main.kt`
- Create: `.../ResearchEvidenceReader.kt`
- Create: `.../EvidenceNormalizer.kt`
- Create: `.../VariantCompiler.kt`
- Create: `.../PackWriter.kt`
- Test: `tools/evidence-pack-compiler/src/test/.../EvidenceCompilerTest.kt`

**Interfaces:**
- Consumes exported `ghidracarista` JSON files through an explicit input directory.
- Produces normalized runtime pack and rejection report.

- [ ] **Step 1:** Add test input with two exact variants for the same feature, one partial variant, one protocol conflict and one missing applicability rule.
- [ ] **Step 2:** Run compiler tests; expected FAIL.
- [ ] **Step 3:** Implement import adapters that map external evidence fields into canonical models without defaulting missing fields.
- [ ] **Step 4:** Implement rejection states `PARTIAL`, `UNRESOLVED`, `CONFLICT`, `PROTECTED`.
- [ ] **Step 5:** Ensure multiple exact variants remain separate and no array-order priority is emitted.
- [ ] **Step 6:** Generate deterministic JSON (stable ordering) and payload hash.
- [ ] **Step 7:** Run tests twice and assert byte-identical output.
- [ ] **Step 8:** Commit `feat(evidence): compile deterministic runtime packs from research exports`.

### Task 5: Evidence invariant validators and coverage skeleton

**Files:**
- Create tools: `schema-validator`, `provenance-validator`, `variant-validator`, `applicability-validator`, `protocol-consistency-validator`, `coverage-report`
- Create: `core-evidence/src/test/.../EvidenceInvariantTest.kt`

**Interfaces:**
- Produces machine-readable validation report and coverage summary.

- [ ] **Step 1:** Add failing fixtures for protocol mismatch, executable ambiguous variants, broken source hash, non-EXACT write, unknown codec.
- [ ] **Step 2:** Implement validators with relationship-based assertions only.
- [ ] **Step 3:** Implement coverage states: CATALOG_ONLY, VARIANT_DISCOVERED, READ_READY, WRITE_READY, RUNTIME_IMPLEMENTED, TRACE_VALIDATED, VEHICLE_VALIDATED, PROTECTED, BLOCKED_EVIDENCE.
- [ ] **Step 4:** Generate `coverage-summary.json` and `coverage-summary.md` from fixtures.
- [ ] **Step 5:** Run all foundation tests and CI-equivalent commands.
- [ ] **Step 6:** Commit `feat(evidence): add invariant validation and coverage reporting`.

### Gate
Do not proceed until `./gradlew check assembleDebug` passes and invalid evidence fixtures are demonstrably rejected.
