# Error Model

**Status:** NORMATIVE. Module `:core-common`, package `com.guns96x.autodiag.core.common.error`.

## 1. Rules

1. Every public suspend API that can fail returns `Outcome<T>`. It never throws for expected failures.
2. `kotlinx.coroutines.CancellationException` is never caught-and-wrapped. Code that uses `runCatching` must rethrow `CancellationException` first (helper `suspendRunCatching` in `core-common` does this).
3. Unexpected exceptions (programming errors) are converted at module boundaries into `AutoDiagError.Internal.Unexpected` with the exception class name and message; the stack trace goes to the engineering log only.
4. Every error carries a stable `code` string. Codes are part of the persisted data contract (Room, traces). They never change once released.
5. `retryable` states whether the **caller's policy** may re-issue the same operation. It does not authorize retrying writes: write retry is governed only by `WRITE_TRANSACTION_SPEC.md` §6.
6. `visibility` controls whether the UI shows the error. `USER` errors have a string resource `error_<code_lowercase>` in EN and UK. `ENGINEERING` errors are shown only in Developer/Traces and in the generic "technical problem" message `error_generic_engineering`.

## 2. Core types

```kotlin
package com.guns96x.autodiag.core.common

public sealed interface Outcome<out T> {
    public data class Success<out T>(val value: T) : Outcome<T>
    public data class Failure(val error: AutoDiagError) : Outcome<Nothing>
}

public inline fun <T, R> Outcome<T>.map(transform: (T) -> R): Outcome<R>
public inline fun <T, R> Outcome<T>.flatMap(transform: (T) -> Outcome<R>): Outcome<R>
public fun <T> Outcome<T>.getOrNull(): T?
public fun <T> Outcome<T>.errorOrNull(): AutoDiagError?
public suspend inline fun <T> suspendRunCatching(block: suspend () -> T): Outcome<T> // rethrows CancellationException

public enum class ErrorVisibility { USER, ENGINEERING }

public enum class RecoveryAction {
    NONE,               // nothing to do; report
    RETRY_OPERATION,    // caller may repeat the same read-only operation
    RECONNECT,          // tear down and reconnect transport, then re-identify
    REINITIALIZE_ADAPTER,
    RESELECT_VARIANT,   // re-run variant selection with fresh identity
    USER_ACTION,        // user must fix a precondition (ignition, voltage, permission)
    UPDATE_APP,
    REPLACE_RUNTIME_PACK,
    OPEN_RECOVERY,      // write transaction recovery screen
}

public sealed class AutoDiagError(
    public val code: String,
    public val retryable: Boolean,
    public val visibility: ErrorVisibility,
    public val recovery: RecoveryAction,
) {
    public abstract val detail: String   // engineering text, English, never localized
}
```

Each leaf below is a `data class` (or `data object` when it has no fields) extending its family class, which extends `AutoDiagError`. Constructor fields are listed; `detail` is computed from them.

## 3. Hierarchy

Legend: R = retryable, V = visibility (U = USER, E = ENGINEERING), Rec = recovery action.

### 3.1 `TransportError`

| Leaf | Fields | Code | R | V | Rec |
|---|---|---|---|---|---|
| `PermissionDenied` | `permission: String` | `TRANSPORT_PERMISSION_DENIED` | no | U | USER_ACTION |
| `AdapterNotFound` | `targetId: String` | `TRANSPORT_DEVICE_NOT_FOUND` | yes | U | USER_ACTION |
| `ConnectTimeout` | `timeoutMs: Long` | `TRANSPORT_CONNECT_TIMEOUT` | yes | U | RECONNECT |
| `ConnectFailed` | `reason: String` | `TRANSPORT_CONNECT_FAILED` | yes | U | RECONNECT |
| `Disconnected` | `reason: String` | `TRANSPORT_DISCONNECTED` | yes | U | RECONNECT |
| `WriteFailed` | `reason: String` | `TRANSPORT_WRITE_FAILED` | yes | E | RECONNECT |
| `NotConnected` | — | `TRANSPORT_NOT_CONNECTED` | no | E | RECONNECT |
| `BluetoothDisabled` | — | `TRANSPORT_BLUETOOTH_DISABLED` | no | U | USER_ACTION |
| `UnsupportedDevice` | `description: String` | `TRANSPORT_UNSUPPORTED_DEVICE` | no | U | NONE |

### 3.2 `AdapterError`

| Leaf | Fields | Code | R | V | Rec |
|---|---|---|---|---|---|
| `NoPrompt` | `command: String, timeoutMs: Long` | `ADAPTER_NO_PROMPT` | yes | E | REINITIALIZE_ADAPTER |
| `UnrecognizedResponse` | `command: String, raw: String` | `ADAPTER_UNRECOGNIZED_RESPONSE` | yes | E | REINITIALIZE_ADAPTER |
| `CommandRejected` | `command: String` (adapter answered `?`) | `ADAPTER_COMMAND_REJECTED` | no | E | NONE |
| `NotIdentified` | `rawIdentity: String` | `ADAPTER_NOT_IDENTIFIED` | no | U | NONE |
| `CapabilityMissing` | `capability: String, state: String` (enum names from `adapter-api`) | `ADAPTER_CAPABILITY_MISSING` | no | U | NONE |
| `BusError` | `token: String` (`CAN ERROR`, `BUS BUSY`, `BUS INIT: ...ERROR`, `FB ERROR`) | `ADAPTER_BUS_ERROR` | yes | U | RETRY_OPERATION |
| `NoData` | `command: String` | `ADAPTER_NO_DATA` | yes | E | RETRY_OPERATION |
| `BufferFull` | — | `ADAPTER_BUFFER_FULL` | yes | E | REINITIALIZE_ADAPTER |
| `InitFailed` | `step: String, raw: String` | `ADAPTER_INIT_FAILED` | yes | U | REINITIALIZE_ADAPTER |

### 3.3 `ProtocolError`

| Leaf | Fields | Code | R | V | Rec |
|---|---|---|---|---|---|
| `Timeout` | `protocol: ProtocolFamily, phase: String, timeoutMs: Long` | `PROTOCOL_TIMEOUT` | yes | U | RETRY_OPERATION |
| `NegativeResponse` | `protocol: ProtocolFamily, serviceId: Int, nrc: Int, nrcName: String?` | `PROTOCOL_NEGATIVE_RESPONSE` | no | U | NONE |
| `ResponsePendingExceeded` | `serviceId: Int, totalWaitMs: Long` | `PROTOCOL_RESPONSE_PENDING_EXCEEDED` | yes | E | RETRY_OPERATION |
| `MalformedFrame` | `protocol: ProtocolFamily, raw: String, reason: String` | `PROTOCOL_MALFORMED_FRAME` | yes | E | RETRY_OPERATION |
| `SequenceError` | `protocol: ProtocolFamily, expected: Int, actual: Int` | `PROTOCOL_SEQUENCE_ERROR` | yes | E | RETRY_OPERATION |
| `FlowControlOverflow` | — (ISO-TP FS=2) | `PROTOCOL_FLOW_CONTROL_OVERFLOW` | no | E | NONE |
| `UnexpectedResponse` | `expectedPrefix: String, actual: String` | `PROTOCOL_UNEXPECTED_RESPONSE` | no | E | NONE |
| `WrongSession` | `required: String, actual: String` | `PROTOCOL_WRONG_SESSION` | yes | E | RETRY_OPERATION |
| `SecurityRequired` | `level: Int?` | `PROTOCOL_SECURITY_REQUIRED` | no | U | NONE |
| `ChannelSetupFailed` | `protocol: ProtocolFamily, reason: String` | `PROTOCOL_CHANNEL_SETUP_FAILED` | yes | E | RECONNECT |
| `NotExecutable` | `reason: String` (profile field is `BLOCKED_EVIDENCE`) | `PROTOCOL_NOT_EXECUTABLE` | no | U | NONE |

`ProtocolFamily` enum (declared in `core-common`, package `com.guns96x.autodiag.core.common`): `CAN_RAW, ISO_TP, UDS, KWP2000, TP20, OBD2`.

`core-common` cannot see enums declared in higher modules. Error fields that refer to such enums (`AdapterCapability`, `CapabilityState`, `IdentityField`, `EvidenceStatus`, `PackViolation`) are therefore typed `String` and hold the enum constant `name`.

NRC mapping is done in `UdsNrc` / `KwpNegativeResponse`: `0x33` → `SecurityRequired`; `0x78` is never surfaced as an error (it extends the wait, see `STATE_MACHINES.md` §5); every other NRC → `NegativeResponse` with the raw value preserved. `nrcName` is non-null only for the codes listed in `API_CONTRACTS.md` §9.4; otherwise `null`.

### 3.4 `DiscoveryError`

| Leaf | Fields | Code | R | V | Rec |
|---|---|---|---|---|---|
| `GatewayUnreachable` | `protocol: ProtocolFamily` | `DISCOVERY_GATEWAY_UNREACHABLE` | yes | U | RETRY_OPERATION |
| `GatewayListMalformed` | `raw: String, reason: String` | `DISCOVERY_GATEWAY_LIST_MALFORMED` | no | E | NONE |
| `NoProtocolDetected` | `tried: List<String>` | `DISCOVERY_NO_PROTOCOL` | yes | U | USER_ACTION |
| `EcuUnreachable` | `logicalAddress: Int` | `DISCOVERY_ECU_UNREACHABLE` | yes | U | RETRY_OPERATION |

### 3.5 `IdentityError`

| Leaf | Fields | Code | R | V | Rec |
|---|---|---|---|---|---|
| `FieldUnavailable` | `field: String` (`IdentityField` enum name) | `IDENTITY_FIELD_UNAVAILABLE` | no | E | NONE |
| `IdentityChanged` | `logicalAddress: Int, changedFields: List<String>` | `IDENTITY_CHANGED` | no | U | RESELECT_VARIANT |
| `VinUnavailable` | — | `IDENTITY_VIN_UNAVAILABLE` | no | U | NONE |

### 3.6 `ApplicabilityError`

| Leaf | Fields | Code | R | V | Rec |
|---|---|---|---|---|---|
| `NoMatch` | `featureId: String` | `APPLICABILITY_NO_MATCH` | no | U | NONE |
| `Ambiguous` | `featureId: String, variantIds: List<String>` | `APPLICABILITY_AMBIGUOUS` | no | U | NONE |
| `PartialOnly` | `featureId: String` | `APPLICABILITY_PARTIAL_ONLY` | no | U | NONE |
| `Protected` | `featureId: String, variantId: String` | `APPLICABILITY_PROTECTED` | no | U | NONE |
| `Conflict` | `featureId: String, variantIds: List<String>` | `APPLICABILITY_CONFLICT` | no | U | NONE |
| `IdentityIncomplete` | `featureId: String, missing: List<String>` | `APPLICABILITY_IDENTITY_INCOMPLETE` | yes | U | RETRY_OPERATION |

### 3.7 `EvidenceError`

| Leaf | Fields | Code | R | V | Rec |
|---|---|---|---|---|---|
| `PackSchemaUnsupported` | `found: Int, supported: IntRange` | `EVIDENCE_PACK_SCHEMA_UNSUPPORTED` | no | U | UPDATE_APP |
| `PackCorrupted` | `reason: String` | `EVIDENCE_PACK_CORRUPTED` | no | U | REPLACE_RUNTIME_PACK |
| `PackSignatureInvalid` | — | `EVIDENCE_PACK_SIGNATURE_INVALID` | no | U | REPLACE_RUNTIME_PACK |
| `PackHashMismatch` | `expected: String, actual: String` | `EVIDENCE_PACK_HASH_MISMATCH` | no | U | REPLACE_RUNTIME_PACK |
| `PackInvariantViolated` | `violations: List<String>` (each `PackViolation.code + ": " + PackViolation.path`) | `EVIDENCE_PACK_INVARIANT_VIOLATED` | no | E | REPLACE_RUNTIME_PACK |
| `NotExecutable` | `entityId: String, status: String` (`EvidenceStatus` enum name) | `EVIDENCE_NOT_EXECUTABLE` | no | U | NONE |
| `NoPackInstalled` | — | `EVIDENCE_NO_PACK_INSTALLED` | no | U | REPLACE_RUNTIME_PACK |
| `PackDowngradeRefused` | `installed: String, candidate: String` | `EVIDENCE_PACK_DOWNGRADE_REFUSED` | no | U | NONE |

### 3.8 `SafetyError`

| Leaf | Fields | Code | R | V | Rec |
|---|---|---|---|---|---|
| `LowVoltage` | `measuredMillivolts: Int, requiredMillivolts: Int` | `SAFETY_LOW_VOLTAGE` | yes | U | USER_ACTION |
| `VoltageUnverifiable` | `requiredMillivolts: Int` | `SAFETY_VOLTAGE_UNVERIFIABLE` | no | U | NONE |
| `PreconditionFailed` | `preconditionId: String, observed: String` | `SAFETY_PRECONDITION_FAILED` | yes | U | USER_ACTION |
| `PreconditionUnverifiable` | `preconditionId: String` | `SAFETY_PRECONDITION_UNVERIFIABLE` | no | U | NONE |
| `ExclusiveLockBusy` | `holder: String` | `SAFETY_EXCLUSIVE_LOCK_BUSY` | yes | U | RETRY_OPERATION |
| `TransportUnstable` | `reconnectsInWindow: Int` | `SAFETY_TRANSPORT_UNSTABLE` | yes | U | USER_ACTION |
| `UserConfirmationMissing` | — | `SAFETY_USER_CONFIRMATION_MISSING` | no | E | NONE |
| `UnsupportedAdapterCapability` | `capability: String` | `SAFETY_UNSUPPORTED_ADAPTER_CAPABILITY` | no | U | NONE |

### 3.9 `WriteError`

| Leaf | Fields | Code | R | V | Rec |
|---|---|---|---|---|---|
| `Rejected` | `nrc: Int` | `WRITE_REJECTED` | no | U | NONE |
| `ReadbackMismatch` | `expectedHex: String, actualHex: String` | `WRITE_READBACK_MISMATCH` | no | U | OPEN_RECOVERY |
| `InterruptedUnknownState` | `transactionId: String, lastState: String` | `WRITE_INTERRUPTED_UNKNOWN_STATE` | no | U | OPEN_RECOVERY |
| `BackupFailed` | `reason: String` | `WRITE_BACKUP_FAILED` | no | U | NONE |
| `OriginalReadFailed` | `cause: String` | `WRITE_ORIGINAL_READ_FAILED` | yes | U | RETRY_OPERATION |
| `ValueNotEncodable` | `reason: String` | `WRITE_VALUE_NOT_ENCODABLE` | no | U | NONE |
| `RollbackNotAllowed` | `reasons: List<String>` | `WRITE_ROLLBACK_NOT_ALLOWED` | no | U | OPEN_RECOVERY |
| `RollbackFailed` | `cause: String` | `WRITE_ROLLBACK_FAILED` | no | U | OPEN_RECOVERY |
| `ConcurrentTransaction` | `activeTransactionId: String` | `WRITE_CONCURRENT_TRANSACTION` | yes | U | RETRY_OPERATION |

### 3.10 `StorageError`

| Leaf | Fields | Code | R | V | Rec |
|---|---|---|---|---|---|
| `WriteFailed` | `entity: String, reason: String` | `STORAGE_WRITE_FAILED` | yes | E | RETRY_OPERATION |
| `ReadFailed` | `entity: String, reason: String` | `STORAGE_READ_FAILED` | yes | E | RETRY_OPERATION |
| `ImmutableRecordModification` | `entity: String, id: String` | `STORAGE_IMMUTABLE_MODIFICATION` | no | E | NONE |
| `DiskFull` | — | `STORAGE_DISK_FULL` | no | U | USER_ACTION |
| `ExportFailed` | `reason: String` | `STORAGE_EXPORT_FAILED` | yes | U | RETRY_OPERATION |

### 3.11 `Internal`

| Leaf | Fields | Code | R | V | Rec |
|---|---|---|---|---|---|
| `Unexpected` | `exceptionClass: String, message: String?` | `INTERNAL_UNEXPECTED` | no | E | NONE |
| `InvariantViolation` | `description: String` | `INTERNAL_INVARIANT_VIOLATION` | no | E | NONE |

## 4. Mapping to master spec §26 failure classes

| Spec failure class | Error leaf |
|---|---|
| ADAPTER_DISCONNECTED | `TransportError.Disconnected` |
| TRANSPORT_TIMEOUT | `TransportError.ConnectTimeout` or `AdapterError.NoPrompt` |
| PROTOCOL_TIMEOUT | `ProtocolError.Timeout` |
| NEGATIVE_RESPONSE | `ProtocolError.NegativeResponse` |
| WRONG_SESSION | `ProtocolError.WrongSession` |
| SECURITY_REQUIRED | `ProtocolError.SecurityRequired` |
| LOW_VOLTAGE | `SafetyError.LowVoltage` |
| PRECONDITION_FAILED | `SafetyError.PreconditionFailed` |
| AMBIGUOUS_VARIANT | `ApplicabilityError.Ambiguous` |
| EVIDENCE_NOT_EXECUTABLE | `EvidenceError.NotExecutable` |
| WRITE_REJECTED | `WriteError.Rejected` |
| READBACK_MISMATCH | `WriteError.ReadbackMismatch` |
| ECU_IDENTITY_CHANGED | `IdentityError.IdentityChanged` |
| UNSUPPORTED_ADAPTER_CAPABILITY | `SafetyError.UnsupportedAdapterCapability` |

## 5. Tests required (Task F2)

`core-common/src/test/kotlin/com/guns96x/autodiag/core/common/error/AutoDiagErrorTest.kt`:

1. `every leaf has a unique code` — reflection-free: a hand-maintained `ALL_ERROR_SAMPLES` list in the test containing one instance of every leaf; assert `codes.size == codes.toSet().size` and that the count of samples equals the count of leaves declared in this document (the test list is the check; add a leaf ⇒ add a sample).
2. `suspendRunCatching rethrows CancellationException`.
3. `Outcome map and flatMap preserve Failure`.
4. `every USER error code has EN and UK string resources` lives in `:ui` (`ErrorStringsTest`, Task S4).
