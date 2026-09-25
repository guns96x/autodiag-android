# API Contracts

**Status:** NORMATIVE. Gemini implements these public contracts exactly unless a later ADR changes them.

All public functions return `Outcome<T>` for expected operational failures. Programming invariant violations are test failures, not runtime control flow.

## 1. Transport

```kotlin
public interface Transport {
    public val state: StateFlow<TransportState>
    public val incoming: Flow<ByteArray>
    public suspend fun connect(target: TransportTarget): Outcome<Unit>
    public suspend fun disconnect(): Outcome<Unit>
    public suspend fun write(data: ByteArray): Outcome<Unit>
    public suspend fun capabilities(): Outcome<TransportCapabilities>
}
```

Ownership: created by `TransportFactory`, owned by a vehicle session, closed when that session ends.  
Dispatcher: all blocking I/O uses `DispatcherProvider.io`.  
Cancellation: cancelling connect/write closes only the in-flight operation; explicit session cancellation closes the transport.

## 2. VehicleAdapter

```kotlin
public interface VehicleAdapter {
    public val state: StateFlow<AdapterState>
    public suspend fun initialize(): Outcome<AdapterIdentity>
    public suspend fun probeCapabilities(): Outcome<AdapterCapabilities>
    public suspend fun configure(profile: AdapterProtocolProfile): Outcome<Unit>
    public suspend fun transmit(frame: AdapterFrame): Outcome<AdapterResponse>
    public suspend fun reset(): Outcome<Unit>
}
```

No automotive service semantics are allowed in adapter implementations.

## 3. DiagnosticChannel

```kotlin
public interface DiagnosticChannel : AutoCloseable {
    public val protocol: ProtocolFamily
    public val ecuLogicalAddress: Int
    public suspend fun exchange(requestHex: String, timeoutMs: Long): Outcome<DiagnosticResponse>
    public override fun close()
}
```

## 4. DiagnosticExecutor

```kotlin
public interface DiagnosticExecutor {
    public suspend fun open(
        ecu: EcuIdentity,
        profile: ProtocolProfile,
        correlation: TraceCorrelation,
    ): Outcome<DiagnosticChannel>

    public suspend fun execute(
        channel: DiagnosticChannel,
        plan: OperationPlan,
        registers: MutableMap<String, String> = mutableMapOf(),
    ): Outcome<OperationExecutionResult>
}
```

## 5. VehicleDetector

```kotlin
public interface VehicleDetector {
    public suspend fun detect(adapter: VehicleAdapter, pack: RuntimePack): Outcome<DetectionReport>
}
```

`DetectionReport` contains one `VehicleIdentity`, discovered `EcuIdentity` entries, unresolved discovery records, and trace session ID.

## 6. EcuDiscoveryEngine

```kotlin
public interface EcuDiscoveryEngine {
    public suspend fun discover(
        adapter: VehicleAdapter,
        pack: RuntimePack,
        candidateProfiles: List<ProtocolProfile>,
    ): Outcome<List<DiscoveredEcu>>
}
```

## 7. EcuIdentityReader

```kotlin
public interface EcuIdentityReader {
    public suspend fun read(
        discovered: DiscoveredEcu,
        pack: RuntimePack,
    ): Outcome<EcuIdentity>
}
```

Missing optional identity fields are `null`; they are never inferred from model name.

## 8. ApplicabilityEngine

```kotlin
public interface ApplicabilityEngine {
    public fun evaluate(
        rule: ApplicabilityRule,
        vehicle: VehicleIdentity,
        ecu: EcuIdentity,
        pack: RuntimePack,
        knownFeatureValues: Map<String, String>,
    ): ApplicabilityResult
}
```

`ApplicabilityResult`: `MATCH`, `NO_MATCH`, `UNKNOWN`, `CONFLICT`.

## 9. VariantSelector

```kotlin
public interface VariantSelector {
    public fun select(
        feature: FeatureDefinition,
        vehicle: VehicleIdentity,
        ecus: List<EcuIdentity>,
        pack: RuntimePack,
        knownFeatureValues: Map<String, String> = emptyMap(),
    ): VariantSelectionResult
}
```

Exact semantics are in `VARIANT_RESOLUTION_SPEC.md`.

## 10. LiveDataScheduler

```kotlin
public interface LiveDataScheduler {
    public fun observe(
        vehicle: VehicleIdentity,
        ecus: List<EcuIdentity>,
        parameters: Set<String>,
        pack: RuntimePack,
    ): Flow<LiveDataSample>

    public suspend fun stop(): Outcome<Unit>
}
```

## 11. CodingRuntime

```kotlin
public interface CodingRuntime {
    public suspend fun read(
        variant: FeatureVariant,
        ecu: EcuIdentity,
        pack: RuntimePack,
    ): Outcome<FeatureReadResult>

    public suspend fun prepareWrite(
        variant: FeatureVariant,
        current: FeatureReadResult,
        requested: DecodedValue,
        pack: RuntimePack,
    ): Outcome<PreparedWrite>
}
```

It does not transmit writes directly. Transmission belongs to `WriteTransactionEngine`.

## 12. AdaptationRuntime

Same contract as `CodingRuntime` with adaptation-specific operation-plan validation:

```kotlin
public interface AdaptationRuntime {
    public suspend fun read(variant: FeatureVariant, ecu: EcuIdentity, pack: RuntimePack): Outcome<FeatureReadResult>
    public suspend fun prepareWrite(
        variant: FeatureVariant,
        current: FeatureReadResult,
        requested: DecodedValue,
        pack: RuntimePack,
    ): Outcome<PreparedWrite>
}
```

## 13. BasicSettingsRuntime / ServiceRuntime

```kotlin
public interface BasicSettingsRuntime {
    public suspend fun execute(
        procedure: ServiceProcedure,
        ecu: EcuIdentity,
        pack: RuntimePack,
    ): Outcome<ServiceExecutionResult>
}

public interface ServiceRuntime {
    public suspend fun execute(
        procedure: ServiceProcedure,
        ecu: EcuIdentity,
        pack: RuntimePack,
    ): Outcome<ServiceExecutionResult>
}
```

Both delegate protocol execution to `DiagnosticExecutor`.

## 14. WriteTransactionEngine

```kotlin
public interface WriteTransactionEngine {
    public suspend fun execute(
        request: WriteRequest,
        vehicle: VehicleIdentity,
        ecus: List<EcuIdentity>,
        pack: RuntimePack,
    ): Outcome<WriteTransaction>
}
```

Exact state and retry rules: `WRITE_TRANSACTION_SPEC.md`.

## 15. RuntimePackLoader / Validator

```kotlin
public interface RuntimePackLoader {
    public suspend fun load(source: RuntimePackSource): Outcome<RuntimePack>
    public suspend fun activate(pack: RuntimePack): Outcome<Unit>
    public suspend fun current(): Outcome<RuntimePack>
}

public interface RuntimePackValidator {
    public fun validate(manifestBytes: ByteArray, payloadBytes: ByteArray): RuntimePackValidationResult
}
```

Activation is atomic: an invalid new pack never replaces the last valid pack.

## 16. TraceRecorder

```kotlin
public interface TraceRecorder : TraceSink {
    public fun openSession(kind: TraceSessionKind): String
    public suspend fun closeSession(sessionId: String): Outcome<Unit>
    public fun events(sessionId: String): Flow<TraceEvent>
}
```

Trace schema: `TRACE_LOGGING_CONTRACT.md`.

## 17. Store ports

Persistence interfaces live in `:core-model`; implementations live in `:storage`. UI and protocol modules never depend on Room directly.
