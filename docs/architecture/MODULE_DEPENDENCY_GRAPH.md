# Module Dependency Graph

**Status:** NORMATIVE. Machine-readable twin: `docs/architecture/module-dependencies.json` (the JSON wins if the two ever disagree; the build task `verifyModuleBoundaries` reads the JSON).

## 1. Changes against master spec §5

The master spec §5 layout is extended by three modules. The reasons are recorded in ADR-0010.

| Added module | Why |
|---|---|
| `:core-common` | Holds the error hierarchy, `Outcome`, clock, dispatchers, trace-event types and `PayloadLink`. Every layer needs them, and `transport-api` must not depend on `core-model`. |
| `:usecase` | Application use cases orchestrate detection, diagnostics, features, safety and services. `core-runtime` sits below the runtimes and therefore cannot host them. |
| `:tools:trace-toolchain` | Protocol decoders and trace replay need protocol modules. The evidence toolchain must not depend on protocols. |

The 16 tool names in master spec §10 are **subcommands** of `:tools:evidence-toolchain` (evidence/pack/coverage tools) and `:tools:trace-toolchain` (trace/decoder/golden-vector tools), not separate Gradle modules. Mapping in §4.

## 2. Layer diagram

```
L0  core-common
L1  core-model   transport-api   protocol-can
L2  core-evidence   adapter-api   protocol-isotp   protocol-uds   protocol-kwp   protocol-tp20
    transport-bluetooth-spp   transport-ble   transport-usb   oem-vag   feature-runtime
L3  adapter-elm   core-runtime   storage
L4  adapter-stn   adapter-vlinker   ecu-discovery   diagnostics-runtime   live-data-runtime
    coding-runtime   adaptation-runtime   safety-runtime
L5  vehicle-detection   basic-settings-runtime   service-runtime
L6  usecase
L7  ui
L8  app
side: testkit (test-only), tools:evidence-toolchain, tools:trace-toolchain
```

An edge may only point to a lower layer.

## 3. Modules

For each module: responsibility, public API (types other modules may use), allowed dependencies (project modules), forbidden dependencies (explicit list of the most likely mistakes; everything not allowed is forbidden).

### `:core-common` — `com.guns96x.autodiag.core.common`
- **Responsibility:** cross-cutting primitives with no domain knowledge.
- **Public API:** `Outcome<T>`, `AutoDiagError` hierarchy (`ERROR_MODEL.md`), `AutoDiagClock`, `DispatcherProvider`, `Hex` (hex encode/decode), `PayloadLink`, `ExchangeTiming`, `TraceEvent`, `TraceSink`, `TraceDirection`, `LogCategory`, `AutoDiagLogger`, `CorrelationId`.
- **Allowed:** none.
- **Forbidden:** every project module.

### `:core-model` — `com.guns96x.autodiag.core.model`
- **Responsibility:** immutable serializable domain types and persistence ports.
- **Public API:** all types in `DATA_MODEL_CONTRACT.md`; ports in package `com.guns96x.autodiag.core.model.store`: `VehicleStore`, `ScanStore`, `LiveDataStore`, `BackupStore`, `WriteTransactionStore`, `OperationHistoryStore`, `TraceStore`, `RuntimePackStore`, `ConnectionProfileStore`.
- **Allowed:** `:core-common`.
- **Forbidden:** `:core-evidence`, `:core-runtime`, any `protocol-*`, any `transport-*`, `:storage`.

### `:core-evidence` — `com.guns96x.autodiag.core.evidence`
- **Responsibility:** runtime-pack parsing, signature/hash verification, fail-closed validation, evidence-status rules.
- **Public API:** `RuntimePackLoader`, `RuntimePackValidator`, `RuntimePackValidationResult`, `PackSignatureVerifier`, `CanonicalJson`, `EvidenceRules`.
- **Allowed:** `:core-common`, `:core-model`.
- **Forbidden:** `:feature-runtime`, `:storage`, `:oem-vag`, any `protocol-*`.

### `:transport-api` — `com.guns96x.autodiag.transport.api`
- **Responsibility:** byte-level transport contract.
- **Public API:** `Transport`, `TransportTarget`, `TransportState`, `TransportCapabilities`, `TransportKind`, `TransportFactory`, `DiscoveredDevice`, `DeviceScanner`.
- **Allowed:** `:core-common`.
- **Forbidden:** `:core-model`, `:adapter-*`, `:protocol-*`, `:oem-vag`, `:feature-runtime`, `:ui`.

### `:transport-bluetooth-spp` — `com.guns96x.autodiag.transport.spp`
- **Responsibility:** Bluetooth Classic RFCOMM/SPP transport.
- **Public API:** `BluetoothSppTransport`, `BluetoothSppScanner`, `BluetoothPermissionGate`.
- **Allowed:** `:core-common`, `:transport-api`.
- **Forbidden:** `:adapter-*` (no AT parsing here), `:protocol-*`.

### `:transport-ble` — `com.guns96x.autodiag.transport.ble`
- **Responsibility:** BLE GATT transport driven by `BleProfile`.
- **Public API:** `BleTransport`, `BleScanner`, `BleProfile`, `BleProfiles`, `BleChunker`.
- **Allowed:** `:core-common`, `:transport-api`.
- **Forbidden:** `:adapter-*`, `:protocol-*`.

### `:transport-usb` — `com.guns96x.autodiag.transport.usb`
- **Responsibility:** USB Host serial transport (CDC-ACM and FTDI FT232R-class only).
- **Public API:** `UsbTransport`, `UsbDeviceMatcher`, `UsbPermissionGate`, `UsbSerialDriver` (internal implementations `CdcAcmDriver`, `FtdiDriver`).
- **Allowed:** `:core-common`, `:transport-api`.
- **Forbidden:** `:adapter-*`, `:protocol-*`.

### `:protocol-can` — `com.guns96x.autodiag.protocol.can`
- **Responsibility:** CAN frame model and identifier validation.
- **Public API:** `CanFrame`, `CanId`, `CanIdFormat`, `CanFrameChannel`.
- **Allowed:** `:core-common`.
- **Forbidden:** `:adapter-*`, `:oem-vag`, `:core-model`.

### `:adapter-api` — `com.guns96x.autodiag.adapter.api`
- **Responsibility:** adapter contract, capability model, AT-dialect framing shared by ELM-family engines.
- **Public API:** `VehicleAdapter`, `AdapterCapabilities`, `CapabilityState`, `AdapterCapability`, `AdapterIdentity`, `AdapterProtocolProfile`, `AdapterFrame`, `AdapterResponse`, `AtCommandCodec`, `PromptFramer`, `AdapterFamily`.
- **Allowed:** `:core-common`, `:transport-api`, `:protocol-can`.
- **Forbidden:** `:protocol-isotp`, `:protocol-uds`, `:protocol-kwp`, `:protocol-tp20`, `:oem-vag`.

### `:adapter-elm` / `:adapter-stn` / `:adapter-vlinker`
- Packages: `com.guns96x.autodiag.adapter.elm`, `.adapter.stn`, `.adapter.vlinker`.
- **Responsibility:** dialect engines. `adapter-stn` and `adapter-vlinker` extend `ElmAdapter`.
- **Public API:** `ElmAdapter`, `StnAdapter`, `VlinkerAdapter`, `ElmCapabilityProber`, `AdapterFactory` implementations.
- **Allowed:** see JSON.
- **Forbidden:** `:protocol-isotp|uds|kwp|tp20`, `:oem-vag`, `:core-model`.

### `:protocol-isotp` — `com.guns96x.autodiag.protocol.isotp`
- **Responsibility:** host-side ISO 15765-2 segmentation/reassembly over a `CanFrameChannel`.
- **Public API:** `IsoTpLink` (implements `PayloadLink`), `IsoTpConfig`, `IsoTpEncoder`, `IsoTpDecoder`.
- **Allowed:** `:core-common`, `:protocol-can`.
- **Forbidden:** `:protocol-uds`, `:adapter-*`, `:oem-vag`.

### `:protocol-uds` — `com.guns96x.autodiag.protocol.uds`
- **Responsibility:** ISO 14229 request/response model, NRC normalization, session and tester-present state machine over a `PayloadLink`.
- **Public API:** `UdsClient`, `UdsSession`, `UdsRequest`, `UdsResponse`, `UdsNrc`, `UdsSessionType`, `UdsTiming`, `SecurityKeyProvider`.
- **Allowed:** `:core-common`.
- **Forbidden:** `:protocol-isotp` (it receives a `PayloadLink`, never constructs one), `:oem-vag`, `:core-model`.

### `:protocol-kwp` — `com.guns96x.autodiag.protocol.kwp`
- **Responsibility:** KWP2000 request/response model and negative-response normalization over a `PayloadLink`.
- **Public API:** `KwpClient`, `KwpRequest`, `KwpResponse`, `KwpNegativeResponse`, `KwpTiming`.
- **Allowed:** `:core-common`.
- **Forbidden:** `:protocol-tp20`, `:oem-vag`.

### `:protocol-tp20` — `com.guns96x.autodiag.protocol.tp20`
- **Responsibility:** TP2.0 channel lifecycle and block codec over a `CanFrameChannel`, exposed as a `PayloadLink`.
- **Public API:** `Tp20Link`, `Tp20ChannelConfig`, `Tp20BlockCodec`, `Tp20Opcode`, `Tp20Timing`.
- **Allowed:** `:core-common`, `:protocol-can`.
- **Forbidden:** `:protocol-kwp`, `:oem-vag`.

### `:core-runtime` — `com.guns96x.autodiag.core.runtime`
- **Responsibility:** vehicle session, bus arbitration, protocol-stack assembly, diagnostic executor, operation-plan interpreter, trace recorder.
- **Public API:** `VehicleSession`, `VehicleSessionFactory`, `BusArbiter`, `ExclusiveOperationLock`, `DiagnosticChannel`, `DiagnosticChannelFactory`, `DiagnosticExecutor`, `OperationPlanInterpreter`, `TraceRecorder`, `ValueAccessStrategy`, `CodecEngine`.
- **Allowed:** see JSON.
- **Forbidden:** `:oem-vag`, `:feature-runtime`, `:storage`, `:ui`.

### `:oem-vag` — `com.guns96x.autodiag.oem.vag`
- **Responsibility:** VAG-specific interpretation: gateway-list record parsing, identity-field mapping, UDS 11-bit address rule cross-check, ECU display naming. All VAG data values (addresses, DIDs) come from the runtime pack. This module holds algorithms, never hard-coded tables.
- **Public API:** `VagGatewayListParser`, `VagIdentityMapper`, `VagAddressRule`, `VagEcuNaming`, `VagOemDescriptor`.
- **Allowed:** `:core-common`, `:core-model`.
- **Forbidden:** `:core-runtime`, any `protocol-*`, `:feature-runtime`.

### `:ecu-discovery` — `com.guns96x.autodiag.discovery`
- **Public API:** `EcuDiscoveryEngine`, `DiscoveredEcu`, `DiscoveryReport`.
- **Allowed:** `:core-common`, `:core-model`, `:core-runtime`, `:oem-vag`.

### `:vehicle-detection` — `com.guns96x.autodiag.detection`
- **Public API:** `VehicleDetector`, `EcuIdentityReader`, `DetectionProgress`, `DetectionReport`.
- **Allowed:** see JSON.

### `:diagnostics-runtime` — `com.guns96x.autodiag.diagnostics`
- **Public API:** `AutoScanEngine`, `DtcReader`, `DtcClearer`, `ScanComparator`.
- **Allowed:** `:core-common`, `:core-model`, `:core-runtime`, `:oem-vag`.

### `:live-data-runtime` — `com.guns96x.autodiag.livedata`
- **Public API:** `LiveDataScheduler`, `LiveDataDecoder`, `LiveDataSubscription`.
- **Allowed:** `:core-common`, `:core-model`, `:core-runtime`.

### `:feature-runtime` — `com.guns96x.autodiag.feature`
- **Responsibility:** applicability evaluation and deterministic variant selection. Pure functions; no I/O.
- **Public API:** `ApplicabilityEngine`, `VariantSelector`, `VariantSelectionResult`, `SelectionExplanation`.
- **Allowed:** `:core-common`, `:core-model`, `:core-evidence`.
- **Forbidden:** `:core-runtime`, any `protocol-*`, `:safety-runtime`, `:storage`.

### `:coding-runtime` — `com.guns96x.autodiag.coding`
- **Public API:** `CodingRuntime` (implements `ValueAccessStrategy` for coding variants), `CodingBlock`, `CodingMutation`.
- **Allowed:** `:core-common`, `:core-model`, `:core-runtime`.

### `:adaptation-runtime` — `com.guns96x.autodiag.adaptation`
- **Public API:** `AdaptationRuntime` (implements `ValueAccessStrategy`), `UdsAdaptationExecutor`, `Tp20AdaptationExecutor`.
- **Allowed:** `:core-common`, `:core-model`, `:core-runtime`.

### `:safety-runtime` — `com.guns96x.autodiag.safety`
- **Public API:** `WriteTransactionEngine`, `SafetyGate`, `PreconditionEvaluator`, `RollbackEvaluator`, `WriteRequest`, `WriteTransactionEvent`.
- **Allowed:** `:core-common`, `:core-model`, `:core-evidence`, `:core-runtime`, `:feature-runtime`.
- **Forbidden:** `:coding-runtime`, `:adaptation-runtime` in main (strategies are injected through `ValueAccessStrategy`), `:storage`, `:ui`.

### `:basic-settings-runtime` — `com.guns96x.autodiag.basicsettings` / `:service-runtime` — `com.guns96x.autodiag.service`
- **Public API:** `BasicSettingsRuntime`; `ServiceRuntime`, `RoutinePoller`.
- **Allowed:** `:core-common`, `:core-model`, `:core-runtime`, `:safety-runtime`.

### `:storage` — `com.guns96x.autodiag.storage`
- **Responsibility:** Room database and implementations of every port in `core.model.store`; runtime-pack file store.
- **Public API:** `AutoDiagDatabase`, `Room*Store` classes (one per port), `FileRuntimePackStore`.
- **Allowed:** `:core-common`, `:core-model`, `:core-evidence`.
- **Forbidden:** every runtime module, `:ui`, `:usecase`.

### `:usecase` — `com.guns96x.autodiag.usecase`
- **Public API:** use-case classes listed in `API_CONTRACTS.md` §19.
- **Allowed:** see JSON. **Forbidden:** `:storage` (uses ports), transport implementations, adapter implementations, `:ui`.

### `:ui` — `com.guns96x.autodiag.ui`
- **Responsibility:** Compose screens, ViewModels, navigation, feature renderer.
- **Allowed:** `:core-common`, `:core-model`, `:usecase`.
- **Forbidden:** `:core-runtime`, any `protocol-*`, `:adapter-*`, `:transport-*`, `:storage`, `:safety-runtime`. The UI can only reach the vehicle through `:usecase`.

### `:app` — `com.guns96x.autodiag`
- **Responsibility:** `AutoDiagApplication` (`@HiltAndroidApp`), `MainActivity`, Hilt modules (`DI_GRAPH.md`), manifest, permissions, build types.

### `:testkit` — `com.guns96x.autodiag.testkit`
- **Public API:** `FakeTransport`, `ScriptedAdapter`, `FakeClock`, `ReplayScript`, `ReplayTransport`, `FaultInjector`, `EcuEmulator`, `FixtureLoader`, `TestPacks`, `TestKeys`.

## 4. Tool mapping (master spec §10 and §43)

| Spec tool | Module | Subcommand |
|---|---|---|
| evidence-pack-compiler | `:tools:evidence-toolchain` | `compile` |
| schema-validator | same | `validate-schema` |
| provenance-validator | same | `validate-provenance` |
| variant-validator | same | `validate-variants` |
| applicability-validator | same | `validate-applicability` |
| protocol-consistency-validator | same | `validate-protocol-consistency` |
| ecu-address-validator | same | `validate-ecu-addresses` |
| request-builder-validator | same | `validate-request-builders` |
| response-parser-validator | same | `validate-response-parsers` |
| runtime-pack-generator | same | `compile` (same command; generator is the output stage) |
| catalog-diff | same | `catalog-diff` |
| coverage-report | same | `coverage-report` |
| evidence-provenance-audit | same | `provenance-audit` |
| (all validators at once) | same | `validate-all` |
| pack-inspector | same | `inspect` |
| golden-vector-generator | `:tools:trace-toolchain` | `golden-vectors` |
| trace-replay | same | `replay` |
| ISO-TP/UDS/KWP/TP2 decoders, trace normalizer | same | `decode` |

## 5. Enforcement: `verifyModuleBoundaries`

Task F1 implements the root task `verifyModuleBoundaries` in `build.gradle.kts` (no separate plugin module). Algorithm:

1. Parse `docs/architecture/module-dependencies.json` with `groovy.json.JsonSlurper`.
2. For every subproject, collect `ProjectDependency` entries from all configurations. Configurations whose name contains `test` or `Test` are checked against `allowed + testAllowed`; all others against `allowed`.
3. Fail with message `Module <path> declares forbidden dependency <dep> in configuration <cfg>` on the first violation.
4. Fail if a subproject exists that is not in the JSON, or a JSON module has no subproject.
5. For `:protocol-*` modules, scan `src/main/**/*.kt` for the forbidden strings in `globalRules` (case-insensitive) and fail with the file path and line.
6. For every module except `:app` and `:ui`, fail if any `src/main/**/*.kt` contains `import dagger.`, `import javax.inject.` or `import androidx.compose.`. For every module except `:storage`, fail on `import androidx.room.`.
