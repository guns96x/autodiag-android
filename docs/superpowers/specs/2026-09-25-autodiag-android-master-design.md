# AutoDiag Android — Master Design Specification

**Status:** Design approved for specification  
**Target:** Independent Android automotive diagnostics, service, live-data, coding/adaptation/basic-settings and customization application  
**Primary evidence source:** `guns96x/ghidracarista`  
**Implementation repository:** `guns96x/autodiag-android`  
**Execution model:** Full implementation by Gemini after this specification and the implementation plans are approved.

---

## 1. Product Goal

Build an independent Android automotive diagnostic and configuration application with the goal of maximizing functional parity with the locally evidenced capabilities and behavior exposed by Carista, while using an independent architecture, independent UI, and a strict evidence-driven implementation model.

The application must support:

- vehicle and ECU discovery;
- adapter capability detection;
- ECU identification;
- AutoScan;
- DTC read and clear;
- ECU information;
- live data;
- coding;
- adaptation;
- basic settings;
- service tools;
- customizations;
- safe write transactions with backup and read-back verification;
- history, traces and backups;
- offline-first operation wherever technically possible;
- expandable OEM/platform support without redesigning the core architecture.

The application must not:

- copy Carista branding, visual design, proprietary assets or text;
- bypass Carista subscription, licensing or account systems;
- impersonate Carista backend services;
- bypass SFD or other OEM authorization/security mechanisms;
- fabricate protocol requests, constants, values, applicability or security procedures.

The governing principle is:

> **No executable write operation without exact evidence for the applicable vehicle/ECU variant.**

---

## 2. Success Definition

The project is complete only when all of the following are true:

1. The full available feature catalog has been imported from the evidence repository.
2. All discovered binary/configuration variants are preserved.
3. A canonical feature can resolve to one or more exact platform/ECU variants.
4. Runtime execution never selects an arbitrary first variant.
5. Every executable variant has:
   - exact ECU target;
   - exact protocol;
   - exact read operation;
   - exact write operation where supported;
   - exact value codec;
   - exact applicability rules;
   - exact evidence provenance.
6. Diagnostics are implemented for every confirmed supported protocol family.
7. Confirmed live-data parameters are implemented with parser evidence.
8. Confirmed coding, adaptation, basic-settings and service-tool flows are implemented.
9. No synthetic protocol commands are generated from feature names.
10. No hard-coded “expected coverage” numbers exist in validators.
11. The application remains functional without Carista backend for local/offline-capable features.
12. Protected or unavailable operations are explicitly marked and never silently bypassed.
13. Protocol golden-vector tests pass.
14. Trace-replay tests pass.
15. Real-vehicle acceptance tests pass for representative supported platforms.

---

## 3. Design Principles

### 3.1 Evidence first

Evidence quality controls runtime capability.

Allowed evidence states:

- `EXACT` — executable when all other safety conditions pass.
- `PARTIAL` — visible for research/debug, never writable.
- `UNRESOLVED` — catalog-only; not executable.
- `CONFLICT` — contradictory evidence; blocked.
- `PROTECTED` — technically mapped but requires legitimate external authorization not available to the application.

### 3.2 Feature is not implementation

A user-facing feature is a semantic concept. Its implementation can vary by platform, ECU generation, ASAM/ODX identity, part number, software version or whitelist.

Example:

```
Feature: car_setting_alarm
  Variant A -> PQ35 -> 46 Central Convenience -> TP2 long coding
  Variant B -> PQ/MK6 -> 09 Central Electronics -> UDS coding
  Variant C -> MQB/MK8 -> 09 Central Electronics -> UDS adaptation
  Variant D -> MEB -> Gateway/other ECU -> UDS adaptation
```

Therefore the runtime model is always:

```
Feature -> Variant[]
```

Never:

```
Feature -> first matching command
```

### 3.3 Protocol logic is reusable, feature data is declarative

Protocol engines implement reusable transport/session/encoding behavior.

Feature packs contain data:

- target ECU;
- applicability;
- read/write plan;
- interpretation;
- safety requirements;
- evidence.

Feature-specific protocol code is only allowed when the operation genuinely cannot be represented by the generic operation model.

### 3.4 Safety before convenience

All writes use a transaction engine with evidence, backup, validation and read-back.

### 3.5 Offline first

All functionality that does not technically require remote authorization must work without cloud services.

---

## 4. High-Level Architecture

```
Android UI
   |
Application / Use Cases
   |
Feature Runtime ---------------- Service Runtime
   |                                  |
Applicability Engine              Operation Planner
   |                                  |
Coding Runtime / Adaptation / Basic Settings / Live Data / Diagnostics
   |
Protocol Engines
   |-- UDS
   |-- KWP2000
   |-- TP2.0
   |-- ISO-TP
   |-- CAN
   |
Adapter Layer
   |-- ELM327
   |-- STN
   |-- vLinker
   |
Transport Layer
   |-- Bluetooth Classic SPP
   |-- BLE
   |-- USB Host / Serial
   |
Vehicle Adapter
```

Independent support services:

```
Evidence Pack Compiler
Runtime Pack Loader
Trace Recorder
Trace Replay
History / Backup Storage
Safety Engine
Coverage Reporter
Validation Toolchain
```

---

## 5. Repository / Module Layout

Target repository structure:

```
autodiag-android/
├── app/
├── core-model/
├── core-evidence/
├── core-runtime/
├── transport-api/
├── transport-bluetooth-spp/
├── transport-ble/
├── transport-usb/
├── adapter-api/
├── adapter-elm/
├── adapter-stn/
├── adapter-vlinker/
├── protocol-can/
├── protocol-isotp/
├── protocol-uds/
├── protocol-kwp/
├── protocol-tp20/
├── vehicle-detection/
├── ecu-discovery/
├── diagnostics-runtime/
├── live-data-runtime/
├── feature-runtime/
├── coding-runtime/
├── adaptation-runtime/
├── basic-settings-runtime/
├── service-runtime/
├── safety-runtime/
├── storage/
├── oem-vag/
├── ui/
├── testkit/
├── tools/
└── docs/
```

### Module responsibility rules

- `transport-*` knows only device I/O.
- `adapter-*` knows adapter command dialect/capabilities.
- `protocol-*` knows automotive protocol state machines.
- `oem-vag` knows VAG identity/address/applicability semantics.
- `feature-runtime` resolves semantic features to exact variants.
- `safety-runtime` controls writes.
- UI never sends raw protocol bytes directly.
- Evidence compiler never depends on Android UI code.

---

## 6. Technology Stack

Mandatory baseline:

- Kotlin
- Jetpack Compose
- Kotlin Coroutines
- Flow / StateFlow
- ViewModel
- Room
- kotlinx.serialization
- Hilt
- Android Bluetooth APIs
- Android USB Host API
- Gradle Kotlin DSL
- Gradle Version Catalog
- JUnit
- Turbine
- fake/mock transports
- ktlint
- detekt
- GitHub Actions

Design constraints:

- no reflection-driven protocol logic;
- no dynamic code execution from feature packs;
- runtime packs are data only;
- all feature-pack schemas versioned;
- all parser/encoder behavior implemented by known runtime codec types.

---

## 7. Canonical Domain Model

### 7.1 VehicleIdentity

Fields:

```
vehicleId
vin
manufacturer
platformFamily
model
modelYear
gatewayIdentity
detectedProtocols[]
discoveryTimestamp
evidence[]
```

### 7.2 EcuIdentity

Fields:

```
logicalAddress
name
transportAddress
protocolFamily
partNumber
softwareVersion
hardwareVersion
serialNumber
componentName
asamOdxId
asamOdxVersion
slaveModules[]
rawIdentityFields
```

### 7.3 EcuVariant

An ECU variant is a normalized applicability identity.

Fields:

```
ecuVariantId
ecuIdentityPredicate
platformPredicate
partNumberRules
softwareVersionRules
hardwareVersionRules
asamRules
whitelistRules
protocolProfileId
evidence[]
```

### 7.4 FeatureDefinition

Semantic feature definition:

```
featureId
canonicalKey
displayMetadata
category
description
interactionType
variants[]
evidenceStatus
```

### 7.5 FeatureVariant

Fields:

```
variantId
featureId
ecuSelector
platformSelector
applicabilityRules[]
protocolProfile
readPlan
writePlan
valueCodec
sessionRequirements
securityRequirements
preconditions[]
postconditions[]
backupPolicy
verificationPolicy
rollbackPolicy
evidence[]
status
```

### 7.6 OperationPlan

```
operationPlanId
operationType
steps[]
timeoutPolicy
retryPolicy
negativeResponsePolicy
cancellationPolicy
cleanupSteps[]
evidence[]
```

### 7.7 ProtocolStep

Supported step families:

```
CONNECT
SET_ADAPTER_MODE
OPEN_PROTOCOL_SESSION
CHANGE_DIAGNOSTIC_SESSION
SECURITY_REQUEST
SEND_REQUEST
EXPECT_RESPONSE
PARSE_RESPONSE
SELECT_ADAPTATION_CHANNEL
READ_CURRENT_VALUE
ENCODE_VALUE
WRITE_VALUE
START_ROUTINE
POLL_ROUTINE
TESTER_PRESENT
READ_BACK
COMPARE
CLEANUP
DELAY
ASSERT_PRECONDITION
```

### 7.8 ValueCodec

No arbitrary scripting.

Allowed codec types include:

```
BOOLEAN_BIT
ENUM_MASK
UNSIGNED_INT
SIGNED_INT
SCALED_INT
ASCII
BCD
BYTE_ARRAY
BITFIELD
RANGE
MULTI_BYTE_ENUM
OEM_DEFINED_CODEC
```

Each codec stores explicit:

- byte offset;
- bit offset;
- mask;
- endian;
- signedness;
- scale;
- additive offset;
- legal values;
- unit;
- invalid-value handling.

---

## 8. Evidence Model

### EvidenceRef fields

```
sourceRepository
sourceFile
sourceFunction
sourceAddress
sourceLineRange
artifactHash
extractionToolVersion
evidenceKind
confidence
notes
```

### Evidence kinds

```
NATIVE_CALLSITE
NATIVE_REQUEST_BUILDER
NATIVE_RESPONSE_PARSER
DEX_CLASS
DEX_CALLSITE
RESOURCE_CONFIG
ASSET_DATABASE
TRACE
OFFICIAL_PROTOCOL
CROSS_VALIDATED
```

### EXACT requirements

A writable feature variant is `EXACT` only if all executable fields are proven:

- concrete class/operation type;
- ECU identity;
- protocol;
- request flow;
- response parser;
- applicability;
- value interpretation;
- write semantics;
- read-back semantics;
- required session;
- required security state;
- evidence provenance.

If any required field is missing, the variant cannot be writable `EXACT`.

---

## 9. Evidence Pack Pipeline

```
ghidracarista
   |
raw evidence ingest
   |
schema validation
   |
provenance validation
   |
variant normalization
   |
protocol consistency validation
   |
applicability validation
   |
operation-plan validation
   |
runtime pack compiler
   |
signed/versioned runtime data pack
   |
AutoDiag Android
```

The application must never read arbitrary research JSON directly.

### Runtime pack properties

Every runtime pack has:

```
schemaVersion
packVersion
buildCommit
sourceCommit
sourceArtifactHashes
generatedAt
featureCount
variantCount
evidenceSummary
payloadHash
```

Runtime pack loading is fail-closed if:

- schema version unsupported;
- payload malformed;
- hash invalid;
- references broken;
- executable feature references non-EXACT evidence;
- operation uses unknown codec or step type.

---

## 10. Evidence Toolchain

Mandatory repository tools:

```
tools/evidence-pack-compiler
tools/schema-validator
tools/provenance-validator
tools/variant-validator
tools/applicability-validator
tools/protocol-consistency-validator
tools/ecu-address-validator
tools/request-builder-validator
tools/response-parser-validator
tools/runtime-pack-generator
tools/catalog-diff
tools/coverage-report
tools/evidence-provenance-audit
tools/golden-vector-generator
tools/trace-replay
tools/pack-inspector
```

### evidence-pack-compiler

Input:

- validated exports from `ghidracarista`.

Output:

- normalized runtime pack.

Must reject:

- arbitrary default values;
- unresolved variants marked executable;
- multiple variants with missing applicability discrimination;
- protocol/class/ECU conflicts;
- command bytes without evidence.

### catalog-diff

Compares:

- previous evidence source;
- current evidence source;
- previous runtime pack;
- current runtime pack.

Reports:

- new features;
- removed features;
- changed variants;
- changed applicability;
- changed requests;
- changed response parsers;
- confidence regressions.

### coverage-report

Machine-readable and human-readable report.

Required dimensions:

```
catalog features
discovered variants
exact variants
partial variants
unresolved variants
conflicts
protected variants
runtime executable read
runtime executable write
live-data parsers
service flows
diagnostic operations
missing applicability
missing parsers
missing response semantics
platform coverage
ECU coverage
protocol coverage
```

No hard-coded expected counts.

---

## 11. Transport API

Canonical interface:

```
interface Transport {
    suspend fun connect(target: TransportTarget)
    suspend fun disconnect()
    suspend fun write(data: ByteArray)
    val incoming: Flow<ByteArray>
    val state: StateFlow<TransportState>
    suspend fun capabilities(): TransportCapabilities
}
```

Transport responsibilities:

- physical connection;
- reconnect detection;
- byte transport;
- device lifecycle;
- permissions;
- connection errors.

Transport must not interpret automotive protocol payloads.

---

## 12. Initial Transport Implementations

### Bluetooth Classic SPP

Required:

- device discovery/selection;
- RFCOMM connection;
- SPP UUID support;
- Android permission handling;
- connection timeout;
- reconnect;
- stream framing;
- adapter prompt handling delegated to adapter layer.

### BLE

Required:

- service discovery;
- characteristic selection based on adapter profile;
- notification subscription;
- MTU handling;
- chunking;
- reconnect.

BLE is adapter-profile driven; no assumption that all BLE OBD adapters expose the same characteristics.

### USB

Required:

- Android USB Host;
- serial device discovery;
- permission flow;
- endpoint discovery;
- line coding where required;
- disconnect handling.

---

## 13. Adapter Abstraction

```
interface VehicleAdapter {
    suspend fun initialize()
    suspend fun probeCapabilities(): AdapterCapabilities
    suspend fun configure(profile: AdapterProtocolProfile)
    suspend fun transmit(frame: AdapterFrame): AdapterResponse
    suspend fun reset()
}
```

Adapter families:

- ELM327-compatible;
- STN;
- vLinker.

Adapter capability probing must determine rather than assume:

- supported CAN modes;
- headers;
- flow control;
- extended addressing;
- ISO-TP offload capability;
- raw CAN access;
- K-Line support if present;
- command-set extensions;
- firmware identity;
- known limitations.

Counterfeit or incomplete ELM implementations must be detected where possible and capabilities downgraded.

---

## 14. Protocol Engines

### 14.1 CAN

Responsibilities:

- frame representation;
- 11-bit and 29-bit identifiers;
- filtering;
- receive matching;
- timing.

### 14.2 ISO-TP

Responsibilities:

- SF/FF/CF/FC;
- segmentation;
- reassembly;
- block size;
- STmin;
- sequence numbers;
- timeout handling;
- padding behavior;
- normal/extended addressing where evidenced.

### 14.3 UDS

State machine covers:

- default/extended/programming session model;
- tester present;
- negative responses;
- pending response handling;
- ReadDataByIdentifier;
- WriteDataByIdentifier;
- RoutineControl;
- ClearDiagnosticInformation;
- ReadDTCInformation;
- ECU reset where evidence exists;
- security-access interface without implementing unauthorized bypass.

### 14.4 KWP2000

Responsibilities:

- request/response service model;
- local identifiers;
- session state;
- negative responses;
- timing;
- transport-independent payload handling.

### 14.5 TP2.0

Responsibilities:

- channel setup;
- connection negotiation;
- block sequencing;
- ACK handling;
- keepalive;
- teardown;
- KWP payload carriage;
- timeouts and retries.

TP2.0 and KWP are separate modules.

---

## 15. Vehicle Detection Pipeline

Canonical detection process:

```
Transport connect
-> adapter initialize
-> adapter capability probe
-> protocol probe
-> VIN acquisition where available
-> gateway detection
-> ECU discovery
-> ECU identification
-> platform inference from exact identity evidence
-> protocol profile resolution
-> ECU variant resolution
-> feature applicability resolution
-> available feature set
```

The system must not infer ECU presence solely from vehicle model.

---

## 16. ECU Discovery

Support at minimum all evidenced VAG discovery methods.

Outputs:

```
logical address
transport address
response status
identity availability
protocol family
installed/active flags
```

Discovery must preserve raw response bytes in trace storage.

---

## 17. ECU Identification

Identification engine supports evidence-backed fields including:

- VIN;
- part number;
- software version;
- hardware version;
- serial number;
- component name;
- ASAM/ODX identifier;
- ASAM/ODX revision;
- workshop coding fields;
- slave/submodule identifiers.

Identity queries are protocol/profile dependent.

---

## 18. Applicability Engine

Applicability is evaluated against exact vehicle and ECU identity.

Rule dimensions may include:

```
manufacturer
platform
model family
year range
logical ECU address
ECU part number
hardware version
software version
ASAM/ODX ID
ASAM revision
whitelist
submodule identity
protocol family
feature dependencies
other required coding state
```

Rules support:

- AND;
- OR;
- explicit deny;
- version range;
- wildcard only where evidence explicitly supports it.

If more than one executable variant matches and there is no deterministic evidence-based priority, runtime must block the write as `AMBIGUOUS_VARIANT`.

---

## 19. Feature Variant Selection

Algorithm:

1. Filter by manufacturer/platform.
2. Filter by ECU presence.
3. Filter by ECU identity.
4. Filter by protocol.
5. Filter by whitelist/ASAM/part/SW/HW rules.
6. Evaluate feature preconditions.
7. Rank only by explicit applicability specificity.
8. Require one exact executable match.

Results:

```
EXACT_MATCH
NO_MATCH
AMBIGUOUS
PARTIAL_ONLY
PROTECTED
CONFLICT
```

Never use array order as priority.

---

## 20. Diagnostics Runtime

Features:

- full AutoScan;
- ECU-specific scan;
- DTC read;
- DTC status;
- clear DTC;
- ECU information;
- before/after scan comparison;
- raw protocol trace;
- saved scan report.

DTC clearing must explicitly indicate scope and affected ECUs.

---

## 21. Live Data Runtime

### LiveDataParameter

```
parameterId
ecuSelector
requestPlan
responseSelector
offset
length
signedness
endianness
scale
additiveOffset
unit
validRange
pollGroup
minimumPollInterval
parserEvidence
```

### Scheduler

Must:

- group compatible parameters;
- respect ECU/protocol timing;
- avoid duplicate requests;
- apply backpressure;
- degrade polling rate when adapter/ECU latency increases;
- expose timestamp and freshness;
- stop cleanly on disconnect.

No PID/DID or scaling may be invented from label text.

---

## 22. Coding Runtime

Coding operations use explicit coding source/read/write builders.

Supports:

- long coding;
- short coding;
- submodule coding;
- byte/bit modification;
- value verification.

Canonical flow:

```
resolve exact variant
-> read current coding
-> decode
-> backup raw coding
-> modify only target field
-> encode
-> validate payload
-> write
-> positive response
-> read back
-> compare complete coding block
```

Unrelated bytes must remain identical unless evidence explicitly requires otherwise.

---

## 23. Adaptation Runtime

Adaptation is distinct from coding.

### UDS adaptation

Typical evidence-backed form:

```
read DID
-> decode field
-> backup
-> modify
-> write DID
-> read back
-> compare
```

### TP2/KWP adaptation

Must model the full verified flow, for example:

```
select adaptation channel
-> read adaptation value
-> decode
-> backup
-> modify
-> write adaptation value
-> read back
-> compare
```

The current evidence indicates application-level services such as:

- select channel `31 B9`;
- read adaptation `31 BA`;
- write adaptation `31 BB`.

Exact payload structure remains variant/request-builder driven.

No synthetic `0x21/0x27` adaptation mapping is allowed without stronger direct evidence.

---

## 24. Basic Settings Runtime

Basic settings are modeled as stateful operation plans, not settings.

Required concepts:

- prerequisite checks;
- session establishment;
- optional security state;
- start;
- status polling;
- stop/cancel where supported;
- terminal success/failure;
- cleanup.

---

## 25. Service Runtime

Each service tool has a dedicated `OperationPlan`.

Examples include:

- EPB service;
- DPF regeneration;
- battery registration;
- service reset;
- TPMS operations;
- actuator tests;
- basic settings;
- OEM-specific maintenance routines.

Service tool UI is generated from procedure metadata but procedure behavior is runtime-plan driven.

---

## 26. Write Safety Engine

Every write operation goes through one central transaction engine.

### Mandatory sequence

```
resolve exact variant
-> verify ECU identity
-> verify runtime pack evidence
-> verify applicability
-> verify adapter capability
-> verify transport stability
-> verify voltage if available
-> verify procedure preconditions
-> read original state
-> save backup
-> execute write
-> verify positive response
-> read back
-> compare expected state
-> store result + trace
```

### Failure classes

```
ADAPTER_DISCONNECTED
TRANSPORT_TIMEOUT
PROTOCOL_TIMEOUT
NEGATIVE_RESPONSE
WRONG_SESSION
SECURITY_REQUIRED
LOW_VOLTAGE
PRECONDITION_FAILED
AMBIGUOUS_VARIANT
EVIDENCE_NOT_EXECUTABLE
WRITE_REJECTED
READBACK_MISMATCH
ECU_IDENTITY_CHANGED
UNSUPPORTED_ADAPTER_CAPABILITY
```

### Rollback

Rollback is permitted only when:

- original state was captured;
- exact reverse write is defined;
- protocol state is valid;
- ECU remains reachable;
- evidence states rollback is safe.

Otherwise the application must preserve backup and trace and stop.

---

## 27. Backups

Every writable coding/adaptation feature stores:

```
backupId
vehicleId
ecuIdentitySnapshot
featureId
variantId
rawBefore
decodedBefore
runtimePackVersion
timestamp
evidenceHash
```

Backups are immutable.

---

## 28. Diagnostic Tracing

Every protocol operation can emit a structured trace:

```
timestamp
transport
adapterCommand
txId
rxId
protocol
request
response
elapsedTime
decodedOperation
NRC
sessionState
featureId
variantId
evidenceRef
```

Sensitive values are redacted only where necessary; automotive diagnostic traces remain available for engineering/debug use.

---

## 29. Storage Model

Room entities:

```
VehicleEntity
EcuEntity
ScanSessionEntity
DtcEntity
LiveDataSessionEntity
LiveDataSampleEntity
FeatureBackupEntity
WriteTransactionEntity
OperationHistoryEntity
DiagnosticTraceEntity
RuntimePackEntity
ConnectionProfileEntity
```

Database migrations are mandatory and tested.

---

## 30. UI Information Architecture

Independent UI structure:

```
Garage
  -> Vehicle Overview
      -> Health / Scan
      -> ECUs
      -> Faults
      -> Live Data
      -> Customizations
      -> Service
      -> History
Connection
Settings
Developer / Traces
```

### Feature interaction types

UI supports declarative controls:

- toggle;
- enum selection;
- numeric range;
- numeric input;
- action;
- wizard;
- multi-step service routine;
- read-only value;
- unsupported/protected state.

UI displays evidence/runtime state where useful:

- available;
- unsupported;
- protected;
- ambiguous;
- unavailable due to adapter capability;
- unavailable due to ECU identity.

---

## 31. User Safety UX

Before a write:

- display target ECU;
- current value;
- requested value;
- required ignition/engine state;
- any procedure-specific prerequisite;
- backup status.

After write:

- show verification result;
- preserve transaction history;
- expose recovery data if read-back mismatch occurs.

No generic “success” is shown until read-back verification passes, except operations whose evidence explicitly defines another success criterion.

---

## 32. Offline / Network Boundary

Local/offline:

- adapter communication;
- vehicle identification;
- diagnostics;
- live data;
- coding;
- adaptation;
- service tools that do not require external authorization;
- feature packs already installed;
- history/backups/traces.

Network may be used for:

- app/runtime-pack updates;
- optional documentation;
- optional cloud backup if separately designed;
- legitimate OEM authorization flows if officially supported in future.

Network must never be required merely because the reference application used a backend.

---

## 33. Protected Operations

If an operation requires SFD or another protected mechanism:

Runtime state:

```
PROTECTED_AUTH_REQUIRED
```

The application may:

- detect the protection;
- display requirements;
- integrate an officially authorized mechanism if one is later available.

It must not:

- generate unauthorized tokens;
- emulate protected authorization servers;
- bypass security checks.

---

## 34. Test Architecture

Four mandatory levels.

### Level 1 — Unit tests

Covers:

- codecs;
- applicability;
- variant resolution;
- state machines;
- parsers;
- encoders;
- evidence-pack loading;
- backup/transaction logic.

### Level 2 — Protocol golden vectors

For each confirmed operation:

```
input state
request bytes
expected response
decoded result
negative responses
timeouts
```

### Level 3 — Trace replay

`FakeTransport` replays captured vehicle sessions.

Fixtures include:

```
PQ35_Golf5_BLS
MQB_generic
MEB_generic
adapter_disconnect
low_voltage
negative_response
response_pending
ambiguous_variant
interrupted_write
malformed_runtime_pack
```

Specific fixtures are added only when real/evidenced traces exist.

### Level 4 — Real vehicle acceptance

Representative test matrix by:

- platform;
- protocol;
- ECU type;
- adapter family;
- read/write capability.

Every real write test starts with a read-only confirmation run.

---

## 35. Testkit

`testkit` supplies:

- FakeTransport;
- FakeAdapter;
- fake clock;
- deterministic scheduler;
- protocol trace fixtures;
- ECU emulator interfaces;
- runtime-pack fixtures;
- fault injection.

Fault injection supports:

- frame loss;
- duplicate frame;
- delayed response;
- disconnect;
- NRC;
- malformed response;
- ECU reset;
- read-back mismatch.

---

## 36. CI

GitHub Actions gates:

1. format;
2. lint;
3. detekt;
4. unit tests;
5. protocol golden-vector tests;
6. runtime-pack schema tests;
7. evidence invariant tests;
8. coverage report generation;
9. Android build;
10. APK artifact generation for approved branches/releases.

No release artifact is generated from a failing evidence validation job.

---

## 37. Evidence Validation Invariants

Validators enforce relationships, never hard-coded totals.

Examples:

- every executable variant has evidence;
- every executable write has read-back policy;
- every referenced ECU exists;
- every protocol operation uses a compatible protocol engine;
- every feature with multiple exact variants has deterministic applicability;
- no `PARTIAL`, `UNRESOLVED`, `CONFLICT` variant appears in executable write pack;
- no unknown codec types;
- no broken evidence references;
- no array-order variant selection;
- no synthetic request bytes;
- no protocol disagreement between variant, ECU transport and operation plan.

---

## 38. Runtime Pack Versioning

Version compatibility:

```
app supported schema range
pack schema version
pack semantic version
source evidence commit
```

Loading policy:

- compatible pack: load;
- newer unsupported schema: reject with update requirement;
- older compatible pack: load;
- corrupted pack: reject;
- pack with downgraded evidence: preserve old pack until explicit valid replacement.

---

## 39. OEM Separation

VAG is the first deeply supported OEM.

Core architecture must not contain VAG-specific assumptions.

OEM modules supply:

- ECU naming;
- addressing profiles;
- discovery;
- identity interpretation;
- applicability schema extensions;
- OEM operation builders where generic protocol operations are insufficient.

Future OEM support is added through a new OEM module + evidence pack, not by modifying core protocol architecture.

---

## 40. VAG Scope

VAG module must eventually cover every VAG capability supported by the available evidence set.

Platform generations are treated explicitly, including evidence for:

- PQ-family / TP2/KWP;
- UDS-based later platforms;
- MQB variants;
- MEB variants where present.

No platform is inferred solely from marketing model name.

---

## 41. Feature Coverage Closure

The project cannot claim parity based on UI count.

Coverage is computed from evidence.

For each feature:

```
catalog existence
-> variant inventory
-> applicability completeness
-> read completeness
-> write completeness
-> response parser
-> value codec
-> safety requirements
-> runtime implementation
-> test coverage
-> real-vehicle validation state
```

Coverage states:

```
CATALOG_ONLY
VARIANT_DISCOVERED
READ_READY
WRITE_READY
RUNTIME_IMPLEMENTED
TRACE_VALIDATED
VEHICLE_VALIDATED
PROTECTED
BLOCKED_EVIDENCE
```

---

## 42. Required Coverage Reports

Generate:

- `coverage-summary.json`
- `coverage-summary.md`
- `feature-coverage.csv`
- `variant-coverage.csv`
- `protocol-coverage.md`
- `platform-coverage.md`
- `blocked-evidence.md`

Reports are generated from data, not maintained manually.

---

## 43. Engineering Tooling Plan

Gemini must create and use:

### Static/data tooling

- evidence importer;
- JSON schema generator/validator;
- coverage analyzer;
- conflict detector;
- pack compiler;
- pack diff;
- provenance audit.

### Protocol tooling

- packet/frame visualizer;
- ISO-TP decoder;
- UDS decoder;
- KWP decoder;
- TP2 decoder;
- trace normalizer;
- request/response golden-vector exporter.

### Test tooling

- trace recorder;
- trace replayer;
- ECU fixture builder;
- fault injector;
- deterministic virtual adapter.

### Developer tooling

- runtime pack inspector;
- feature inspector;
- variant resolution debugger;
- ECU identity inspector;
- operation-plan debugger.

---

## 44. Logging

Structured logging categories:

```
TRANSPORT
ADAPTER
CAN
ISOTP
UDS
KWP
TP20
DISCOVERY
IDENTITY
APPLICABILITY
FEATURE
SERVICE
SAFETY
STORAGE
EVIDENCE
```

Production logging must avoid excessive personal data while retaining automotive debugging utility.

---

## 45. Performance Requirements

The app must:

- avoid blocking main thread;
- stream live data using Flow;
- use bounded trace buffers;
- persist long traces asynchronously;
- avoid loading entire evidence packs repeatedly;
- cache resolved vehicle/ECU identity while detecting identity changes;
- maintain responsive UI during AutoScan.

No arbitrary target latency is specified where adapter/vehicle timing dominates; protocol timing requirements take precedence.

---

## 46. Connection Recovery

Connection loss handling:

1. stop active writes immediately;
2. mark transaction interrupted;
3. preserve trace and backup;
4. reconnect only according to operation safety policy;
5. re-identify ECU before continuing;
6. never silently repeat a write unless the operation is explicitly idempotent and evidence permits retry.

---

## 47. Negative Response Handling

UDS/KWP negative responses are normalized into structured error objects while retaining raw NRC/service context.

Response-pending is handled according to protocol timing rather than treated as immediate failure.

Unknown NRCs are preserved raw.

---

## 48. Concurrency

Default policy:

- one active write transaction per vehicle;
- no concurrent service routine and coding write on the same vehicle;
- live-data polling suspends or degrades when an exclusive operation requires it;
- read-only ECU operations may run concurrently only if transport/protocol/adapter permits safe multiplexing.

Concurrency rules are protocol-profile configurable.

---

## 49. Security Model

The application treats automotive security access as an explicit operation dependency.

Allowed:

- known legitimate seed/key provider interfaces;
- user-supplied/official authorization mechanisms;
- detection/reporting of required access level.

Not allowed:

- guessed keys;
- brute force;
- unauthorized security bypass.

---

## 50. Developer / Engineering Mode

Developer mode provides:

- raw trace viewer;
- ECU identity dump;
- variant-resolution explanation;
- runtime-pack metadata;
- exact evidence references;
- operation-plan preview;
- read-only raw request console only where explicitly enabled for engineering builds.

Production write execution remains constrained by the safety engine.

---

## 51. Configuration Import Rules

No research artifact becomes executable simply because it exists.

Import progression:

```
raw
-> parsed
-> normalized
-> provenance-valid
-> conflict-free
-> applicability-complete
-> operation-complete
-> executable
```

Every transition is machine validated.

---

## 52. Gemini Execution Phases

Gemini must implement the complete application in phased, reviewable commits.

### Phase 0 — Repository foundation

- Gradle multi-module project;
- CI;
- lint/static analysis;
- baseline app shell;
- testing infrastructure.

### Phase 1 — Core domain model

Implement all canonical types and schemas.

### Phase 2 — Evidence compiler/toolchain

Implement evidence import, validation, pack generation and coverage reporting.

### Phase 3 — Transport layer

Bluetooth SPP, BLE, USB.

### Phase 4 — Adapter engines

ELM, STN, vLinker capability probing and I/O.

### Phase 5 — CAN + ISO-TP

Full tested protocol support required by the evidence set.

### Phase 6 — UDS

Session, DID, DTC, routine, negative response and tester-present support.

### Phase 7 — KWP + TP2.0

Session transport, TP2 channel, coding/adaptation primitives.

### Phase 8 — Vehicle and ECU detection

Discovery + identity + variant resolution.

### Phase 9 — Diagnostics

AutoScan, DTC and ECU information.

### Phase 10 — Live data

Runtime, scheduler, codecs and UI.

### Phase 11 — Feature/applicability runtime

Feature catalog, variant matching and UI definitions.

### Phase 12 — Coding

Read/modify/write/read-back flow.

### Phase 13 — Adaptation

UDS and TP2/KWP adaptation flows.

### Phase 14 — Basic settings

Stateful routine execution.

### Phase 15 — Service tools

All evidence-backed tools.

### Phase 16 — Safety/write transaction engine

Backups, validation, recovery, read-back.

### Phase 17 — Storage/history

Room models, migrations, history, backups and traces.

### Phase 18 — Complete UI

All product surfaces.

### Phase 19 — Full evidence import

Import all eligible `ghidracarista` evidence and generate complete runtime packs.

### Phase 20 — Coverage closure

Resolve every feature into implemented/blocked/protected state with generated reports.

### Phase 21 — Real-vehicle validation

Representative platforms/adapters/operations.

### Phase 22 — Hardening/release

Performance, recovery, migration, compatibility, release engineering.

---

## 53. Gemini Working Rules

Gemini must:

- work phase by phase;
- run tests before each commit;
- push staged commits;
- never mark unsupported evidence executable;
- never preserve a failing EXACT count for cosmetic reasons;
- prefer downgrade to PARTIAL/CONFLICT over guessing;
- document blockers;
- update generated coverage after material changes;
- maintain reproducibility;
- avoid massive monolithic files;
- preserve module boundaries;
- keep implementation deterministic and testable.

Gemini must not stop merely because some features are blocked by missing evidence. It should complete all implementable architecture/functionality and clearly classify the remainder.

---

## 54. Definition of “100%”

“100%” means:

- 100% of the available catalog accounted for;
- 100% of discovered variants preserved;
- 100% of executable operations backed by exact evidence;
- 100% of feature states machine-classified;
- 100% of implemented write paths safety-gated;
- 100% of supported protocol paths tested with golden vectors;
- no known silent fallback or fabricated mapping.

It does **not** mean fabricating behavior for unsupported/protected/unresolved features.

---

## 55. Non-Goals

This project does not include:

- ECU flashing/chiptuning;
- Carista account/subscription emulation;
- copying Carista UI;
- unauthorized backend access;
- SFD bypass;
- credential extraction;
- arbitrary universal raw-write console in normal user mode.

---

## 56. Final Acceptance Gate

Before declaring the product complete, all of the following artifacts must exist and pass:

- Android release build;
- runtime evidence pack;
- evidence validation report;
- feature coverage report;
- variant coverage report;
- protocol coverage report;
- platform coverage report;
- golden-vector suite;
- trace-replay suite;
- Room migration tests;
- write-safety tests;
- real-vehicle acceptance report;
- unresolved/protected feature report.

The final reviewer must be able to select any implemented feature and trace:

```
UI
-> FeatureDefinition
-> FeatureVariant
-> applicability
-> ECU identity
-> protocol
-> OperationPlan
-> exact request/response behavior
-> ValueCodec
-> write safety transaction
-> evidence provenance
-> tests
```

If any executable path cannot be traced end-to-end, it is not accepted as complete.
