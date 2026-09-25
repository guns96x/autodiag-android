# Data Model Contract

**Status:** NORMATIVE. Types live in `:core-model`, package `com.guns96x.autodiag.core.model` unless stated otherwise. Master spec §7, §8, §21, §26–28 field lists map onto these types (§17 of this document gives the mapping).

## 0. Global conventions

| Topic | Rule |
|---|---|
| Serialization | `kotlinx.serialization`. One shared instance `AutoDiagJson` in `core-model`: `Json { classDiscriminator = "type"; ignoreUnknownKeys = false; explicitNulls = false; encodeDefaults = true; prettyPrint = false }`. |
| Property names | JSON names equal Kotlin property names (camelCase). `@SerialName` is used **only** on sealed subclasses (the `type` discriminator value, listed per type). |
| Immutability | All model classes are `data class` with `val` properties and `List`/`Map` (read-only) collections. |
| IDs | `String`, non-blank, regex `^[a-z0-9][a-z0-9_.:-]{0,127}$`, except where a stricter regex is given. |
| Timestamps | `Long`, epoch milliseconds UTC. Name suffix `At` (e.g. `createdAt`). |
| Durations | `Long` milliseconds, name suffix `Ms`. |
| Bytes in models | `ByteArray` is never used in serializable models. Bytes are `String` lowercase hex without separators (`"22f1a3"`), regex `^([0-9a-f]{2})*$`, name suffix `Hex`. |
| Addresses/IDs of CAN | `Int`. 11-bit range `0x000..0x7FF`, 29-bit range `0x00000000..0x1FFFFFFF`. |
| Validation | Each class with constraints has a companion `fun validate(): List<ModelViolation>`; constructors do not throw. `ModelViolation(path: String, code: String, message: String)`. Validators run in `RuntimePackValidator` and in tests; code never trusts unvalidated instances. |
| Evidence references | Entities reference evidence by `evidenceIds: List<String>`, resolved against `RuntimePack.evidence`. |

## 1. Shared enums

```kotlin
// core-common
public enum class ProtocolFamily { CAN_RAW, ISO_TP, UDS, KWP2000, TP20, OBD2 }

// core-model
@Serializable public enum class EvidenceStatus { EXACT, PARTIAL, UNRESOLVED, CONFLICT, PROTECTED }
@Serializable public enum class EvidenceKind {
    NATIVE_CALLSITE, NATIVE_REQUEST_BUILDER, NATIVE_RESPONSE_PARSER, DEX_CLASS, DEX_CALLSITE,
    RESOURCE_CONFIG, ASSET_DATABASE, TRACE, OFFICIAL_PROTOCOL, CROSS_VALIDATED,
}
@Serializable public enum class EvidenceConfidence { VERIFIED, CORROBORATED, UNVERIFIED }
@Serializable public enum class Manufacturer { VAG, UNKNOWN }
@Serializable public enum class PlatformFamily { PQ25, PQ35, PQ46, MQB, MQB_EVO, MLB, MLB_EVO, MEB, UNKNOWN }
@Serializable public enum class IdentityField {
    VIN, PART_NUMBER, SOFTWARE_VERSION, HARDWARE_VERSION, SERIAL_NUMBER, COMPONENT_NAME,
    ASAM_ODX_ID, ASAM_ODX_VERSION, WORKSHOP_CODE, SLAVE_MODULES,
}
@Serializable public enum class Endianness { BIG, LITTLE }
@Serializable public enum class InteractionType {
    TOGGLE, ENUM_SELECTION, NUMERIC_RANGE, NUMERIC_INPUT, ACTION, WIZARD, SERVICE_ROUTINE, READ_ONLY,
}
@Serializable public enum class FeatureCategory {
    DIAGNOSTICS, LIVE_DATA, CODING, ADAPTATION, BASIC_SETTING, SERVICE, CUSTOMIZATION, INFORMATION,
}
@Serializable public enum class CoverageState {
    CATALOG_ONLY, VARIANT_DISCOVERED, READ_READY, WRITE_READY, RUNTIME_IMPLEMENTED,
    TRACE_VALIDATED, VEHICLE_VALIDATED, PROTECTED, BLOCKED_EVIDENCE,
}
```

## 2. `EvidenceRef`

```kotlin
@Serializable
public data class EvidenceRef(
    val id: String,                    // "ev:" + first 16 hex chars of SHA-256 over the canonical JSON of the other fields
    val registerId: String?,           // e.g. "EV-UDS-02"; null only for per-variant evidence
    val sourceRepository: String,      // "guns96x/ghidracarista" or "standard"
    val sourceCommit: String?,         // 40 hex; null only when evidenceKind == OFFICIAL_PROTOCOL
    val sourceFile: String?,           // repository-relative path; null only for OFFICIAL_PROTOCOL
    val sourceFunction: String?,
    val sourceAddress: String?,        // "0x583306" or "line 104450"
    val sourceLineRange: String?,      // "107134-107148"
    val standardRef: String?,          // required iff evidenceKind == OFFICIAL_PROTOCOL, e.g. "ISO 14229-1:2020 11.2"
    val artifactHash: String?,         // SHA-256 (64 hex) of sourceFile at sourceCommit; null only for OFFICIAL_PROTOCOL
    val extractionToolVersion: String, // "evidence-toolchain <semver>"
    val evidenceKind: EvidenceKind,
    val confidence: EvidenceConfidence,
    val notes: String = "",
)
```

Validation:
- `id` regex `^ev:[0-9a-f]{16}$`.
- If `evidenceKind == OFFICIAL_PROTOCOL`: `standardRef` non-blank; `sourceCommit`, `sourceFile`, `artifactHash` must be null.
- Otherwise: `sourceCommit` regex `^[0-9a-f]{40}$`, `sourceFile` non-blank, `artifactHash` regex `^[0-9a-f]{64}$`, and at least one of `sourceFunction`, `sourceAddress`, `sourceLineRange` non-null.
- `confidence == VERIFIED` is required for any evidence referenced by an `EXACT` executable entity (other than `OFFICIAL_PROTOCOL`, which is `VERIFIED` by definition).

## 3. `VehicleIdentity`

```kotlin
@Serializable
public data class VehicleIdentity(
    val vehicleId: String,                    // "vin:<VIN>" when vin != null, else "anon:<uuid-v4>"
    val vin: String?,                         // regex ^[A-HJ-NPR-Z0-9]{17}$
    val manufacturer: Manufacturer,           // VAG when the pack's OEM descriptor matched, else UNKNOWN
    val platformFamily: PlatformFamily?,      // null unless a pack PlatformRule with EXACT evidence matched
    val model: String?,                       // always null in v1 (no evidence-backed model decoding)
    val modelYear: Int?,                      // always null in v1
    val gatewayLogicalAddress: Int?,          // logical address of the ECU that answered the gateway list
    val detectedProtocols: List<ProtocolFamily>, // sorted by enum ordinal, distinct
    val discoveryTimestamp: Long,
    val evidenceIds: List<String>,
)
```

The spec field `gatewayIdentity` is represented by `gatewayLogicalAddress` plus the gateway's `EcuIdentity` in the same `DetectionReport`.

## 4. `EcuIdentity`

```kotlin
@Serializable
public data class EcuIdentity(
    val logicalAddress: Int,                  // 0x00..0xFF
    val name: String,                         // EcuDefinition.name from the pack, else "ECU_%02X"
    val transportAddress: TransportAddress,
    val protocolFamily: ProtocolFamily,       // UDS or KWP2000
    val partNumber: String?,
    val softwareVersion: String?,
    val hardwareVersion: String?,             // always null on UDS in v1 (EV-UDS-13)
    val serialNumber: String?,
    val componentName: String?,
    val asamOdxId: String?,
    val asamOdxVersion: String?,
    val workshopCode: String?,
    val slaveModules: List<SlaveModuleIdentity>,
    val rawIdentityFields: Map<String, String>, // key = request hex (e.g. "22f187"), value = full positive response hex
    val readAt: Long,
)

@Serializable
public sealed interface TransportAddress {
    @Serializable @SerialName("isotp11") public data class IsoTp11(val txId: Int, val rxId: Int) : TransportAddress
    @Serializable @SerialName("isotp29") public data class IsoTp29(val txId: Int, val rxId: Int) : TransportAddress
    @Serializable @SerialName("tp20") public data class Tp20(val logicalAddress: Int) : TransportAddress
}

@Serializable
public data class SlaveModuleIdentity(val index: Int, val rawHex: String)
```

Text identity fields are decoded as US-ASCII, bytes `0x00` and `0x20` are trimmed from both ends, and non-printable bytes (`< 0x20` or `> 0x7E`) make the field `null` while the raw bytes stay in `rawIdentityFields`.

## 5. Applicability

```kotlin
@Serializable
public sealed interface ApplicabilityRule {
    @Serializable @SerialName("all") public data class All(val rules: List<ApplicabilityRule>) : ApplicabilityRule
    @Serializable @SerialName("any") public data class AnyOf(val rules: List<ApplicabilityRule>) : ApplicabilityRule
    @Serializable @SerialName("not") public data class Not(val rule: ApplicabilityRule) : ApplicabilityRule
    @Serializable @SerialName("ecuAddress") public data class EcuAddress(val logicalAddress: Int) : ApplicabilityRule
    @Serializable @SerialName("protocol") public data class ProtocolIs(val family: ProtocolFamily) : ApplicabilityRule
    @Serializable @SerialName("platform") public data class PlatformIn(val platforms: List<PlatformFamily>) : ApplicabilityRule
    @Serializable @SerialName("fieldEquals") public data class FieldEquals(val field: IdentityField, val value: String) : ApplicabilityRule
    @Serializable @SerialName("fieldIn") public data class FieldIn(val field: IdentityField, val values: List<String>) : ApplicabilityRule
    @Serializable @SerialName("fieldPrefix") public data class FieldPrefix(val field: IdentityField, val prefix: String, val evidenceIds: List<String>) : ApplicabilityRule
    @Serializable @SerialName("versionRange") public data class VersionRange(
        val field: IdentityField, val minInclusive: String?, val maxInclusive: String?, val comparison: VersionComparison,
    ) : ApplicabilityRule
    @Serializable @SerialName("whitelist") public data class InWhitelist(val whitelistId: String) : ApplicabilityRule
    @Serializable @SerialName("featureValue") public data class FeatureValueIs(val featureId: String, val encodedValueHex: String) : ApplicabilityRule
    @Serializable @SerialName("always") public data object Always : ApplicabilityRule
}

@Serializable public enum class VersionComparison { NUMERIC_DECIMAL, ASCII_LEXICOGRAPHIC }

@Serializable
public data class WhitelistDefinition(
    val whitelistId: String,                 // from source symbol, e.g. "vag.wl.central_elec_b8"
    val sourceSymbol: String,                // "VagWhitelists::CENTRAL_ELEC_B8"
    val kind: WhitelistKind,
    val entries: List<String>,               // exact strings, or "min..max" for SOFTWARE_VERSION_RANGE
    val evidenceIds: List<String>,
    val status: EvidenceStatus,              // EXACT only when entries come from committed evidence (EV-CD-06)
)
@Serializable public enum class WhitelistKind { PART_NUMBER, SOFTWARE_VERSION_RANGE, ASAM_ODX_ID, MIXED }
```

Validation: `All`/`AnyOf` have ≥ 1 rule; `FieldIn.values` non-empty and distinct; `FieldPrefix.evidenceIds` non-empty (wildcards only with evidence, spec §18); `VersionRange` has at least one bound; `NUMERIC_DECIMAL` bounds match `^[0-9]+$`. Evaluation semantics: `VARIANT_RESOLUTION_SPEC.md` §3.

## 6. ECU definitions and variants

```kotlin
@Serializable
public data class EcuDefinition(
    val logicalAddress: Int,
    val name: String,                        // e.g. "CENTRAL_ELEC"
    val addressing: List<EcuAddressing>,     // one per protocol family the ECU is evidenced on
)

@Serializable
public data class EcuAddressing(
    val protocolProfileId: String,
    val transportAddress: TransportAddress,
    val evidenceIds: List<String>,
    val status: EvidenceStatus,
)

@Serializable
public data class EcuVariant(
    val ecuVariantId: String,
    val logicalAddress: Int,
    val predicate: ApplicabilityRule,        // spec fields ecuIdentityPredicate, platformPredicate, partNumberRules,
                                             // softwareVersionRules, hardwareVersionRules, asamRules, whitelistRules
                                             // are all expressed inside this single rule tree
    val protocolProfileId: String,
    val evidenceIds: List<String>,
    val status: EvidenceStatus,
)
```

## 7. `ProtocolProfile`

```kotlin
@Serializable
public data class ProtocolProfile(
    val profileId: String,                   // e.g. "vag.uds.isotp11"
    val payloadProtocol: ProtocolFamily,     // UDS, KWP2000 or OBD2
    val link: LinkKind,                      // ISO_TP_11, ISO_TP_29 or TP20
    val requiredAdapterCapabilities: List<String>, // AdapterCapability names, see TRANSPORT_ADAPTER_CONTRACT.md §5
    val timing: ProtocolTiming,
    val session: SessionModel?,              // null for KWP2000/OBD2
    val evidenceIds: List<String>,
    val status: EvidenceStatus,
    val blockedFields: List<String>,         // names of fields whose value is BLOCKED_EVIDENCE; non-empty ⇒ status != EXACT
)

@Serializable public enum class LinkKind { ISO_TP_11, ISO_TP_29, TP20 }

@Serializable
public data class ProtocolTiming(
    val p2Ms: Long,                          // UDS: 50 (EV-UDS-06); KWP/TP20: BLOCKED (0 + listed in blockedFields)
    val p2StarMs: Long,                      // UDS: 5000 (EV-UDS-05)
    val adapterResponseTimeoutMs: Long,      // 5000 (EV-TR-04)
    val extendedServiceTimeoutMs: Long,      // 10000 (EV-TR-04)
)

@Serializable
public data class SessionModel(
    val defaultSession: Int,                 // 0x01
    val extendedSession: Int,                // 0x03
    val testerPresentRequestHex: String,     // "3e80"
    val testerPresentPeriodMs: Long,         // 2000
    val s3Ms: Long,                          // 5000
)
```

Allowed `(payloadProtocol, link)` pairs: `(UDS, ISO_TP_11)`, `(UDS, ISO_TP_29)`, `(OBD2, ISO_TP_11)`, `(KWP2000, TP20)`. Any other pair is `CONFLICT` (`PROTOCOL_CONSISTENCY.md`).

## 8. `ValueCodec`

```kotlin
@Serializable
public sealed interface ValueCodec {
    val codecId: String

    @Serializable @SerialName("booleanBit")
    public data class BooleanBit(override val codecId: String, val byteOffset: Int, val bitMask: Int, val inverted: Boolean) : ValueCodec
    @Serializable @SerialName("enumMask")
    public data class EnumMask(override val codecId: String, val byteOffset: Int, val mask: Int, val options: List<EnumOption>) : ValueCodec
    @Serializable @SerialName("unsignedInt")
    public data class UnsignedInt(override val codecId: String, val byteOffset: Int, val byteLength: Int, val endian: Endianness) : ValueCodec
    @Serializable @SerialName("signedInt")
    public data class SignedInt(override val codecId: String, val byteOffset: Int, val byteLength: Int, val endian: Endianness) : ValueCodec
    @Serializable @SerialName("scaledInt")
    public data class ScaledInt(
        override val codecId: String, val byteOffset: Int, val byteLength: Int, val endian: Endianness, val signed: Boolean,
        val scale: Double, val additiveOffset: Double, val unit: String, val validMin: Double?, val validMax: Double?,
        val invalidRawValues: List<Long>,
    ) : ValueCodec
    @Serializable @SerialName("ascii")
    public data class Ascii(override val codecId: String, val byteOffset: Int, val byteLength: Int?, val trim: Boolean) : ValueCodec
    @Serializable @SerialName("bcd")
    public data class Bcd(override val codecId: String, val byteOffset: Int, val byteLength: Int) : ValueCodec
    @Serializable @SerialName("byteArray")
    public data class ByteArrayCodec(override val codecId: String, val byteOffset: Int, val byteLength: Int?) : ValueCodec
    @Serializable @SerialName("bitfield")
    public data class Bitfield(override val codecId: String, val byteOffset: Int, val bitOffset: Int, val bitLength: Int, val endian: Endianness) : ValueCodec
    @Serializable @SerialName("range")
    public data class Range(
        override val codecId: String, val byteOffset: Int, val byteLength: Int, val endian: Endianness, val signed: Boolean,
        val scale: Double, val additiveOffset: Double, val unit: String, val min: Double, val max: Double, val step: Double,
    ) : ValueCodec
    @Serializable @SerialName("multiByteEnum")
    public data class MultiByteEnum(override val codecId: String, val byteOffset: Int, val byteLength: Int, val endian: Endianness, val options: List<EnumOption>) : ValueCodec
    @Serializable @SerialName("oemDefined")
    public data class OemDefined(override val codecId: String, val oemCodecId: String) : ValueCodec
}

@Serializable public data class EnumOption(val optionId: String, val rawValue: Long, val labelKey: String)
```

Validation (all codecs): `byteOffset >= 0`; `byteLength in 1..8` for integer codecs, `1..255` otherwise; `BooleanBit.bitMask` has exactly one bit set and is in `1..0x80`; `EnumMask.mask in 1..0xFF`, every option's `rawValue` satisfies `rawValue and mask.inv() == 0`, option IDs and raw values distinct, ≥ 2 options; `Bitfield.bitOffset in 0..7`, `bitLength in 1..32`; `Range.min < max`, `step > 0`; `ScaledInt.scale != 0.0`. `OemDefined` is valid only if `oemCodecId` is registered in the OEM codec registry; in v1 the registry is empty, so any `OemDefined` makes its entity non-executable.

Semantics:
- Bit mask is applied to the byte at `byteOffset` of the **value block** (the bytes after the positive-response prefix and DID/channel echo; defined per `ValueTarget` in §10).
- Integers: `raw` is assembled with `endian`; `signed` uses two's complement of `byteLength*8` bits.
- Scaled: `physical = raw * scale + additiveOffset`. Encoding: `raw = round((physical - additiveOffset) / scale)` with `RoundingMode.HALF_EVEN`; encode fails with `WriteError.ValueNotEncodable` if `raw` is outside the representable range or `physical` outside `[validMin, validMax]`/`[min, max]`, or not on the `step` grid (tolerance `1e-9 * step`).
- `BooleanBit`: decoded `true` iff `(byte and bitMask) != 0` xor `inverted`.
- `EnumMask`: `rawValue = byte and mask`; unknown values decode to `DecodedValue.Invalid`.
- Encoding modifies only the addressed bits/bytes; all other bytes of the value block are copied unchanged (spec §22).

```kotlin
@Serializable
public sealed interface DecodedValue {
    @Serializable @SerialName("bool") public data class Bool(val value: Boolean) : DecodedValue
    @Serializable @SerialName("enum") public data class Enum(val optionId: String, val rawValue: Long) : DecodedValue
    @Serializable @SerialName("int") public data class Integer(val value: Long) : DecodedValue
    @Serializable @SerialName("scaled") public data class Scaled(val value: Double, val unit: String) : DecodedValue
    @Serializable @SerialName("text") public data class Text(val value: String) : DecodedValue
    @Serializable @SerialName("bytes") public data class Bytes(val hex: String) : DecodedValue
    @Serializable @SerialName("invalid") public data class Invalid(val rawHex: String, val reason: String) : DecodedValue
}
```

## 9. Operation plans

```kotlin
@Serializable
public data class OperationPlan(
    val operationPlanId: String,
    val operationType: OperationType,
    val protocolProfileId: String,
    val steps: List<ProtocolStep>,
    val cleanupSteps: List<ProtocolStep>,     // always executed after steps, also on failure and cancellation (NonCancellable)
    val timeoutPolicy: TimeoutPolicy,
    val retryPolicy: RetryPolicy,
    val negativeResponsePolicy: NegativeResponsePolicy,
    val cancellationPolicy: CancellationPolicy,
    val evidenceIds: List<String>,
    val status: EvidenceStatus,
)

@Serializable public enum class OperationType {
    IDENTIFY, READ_DTC, CLEAR_DTC, READ_VALUE, WRITE_VALUE, READ_LIVE_DATA, BASIC_SETTING, SERVICE_ROUTINE, DISCOVERY,
}

@Serializable public data class TimeoutPolicy(val stepTimeoutMs: Long, val totalTimeoutMs: Long)
@Serializable public data class RetryPolicy(val maxAttempts: Int, val backoffMs: Long, val retryOn: List<String>) // retryOn = error codes
@Serializable public data class NegativeResponsePolicy(val treatAsSuccess: List<Int>, val abortOn: List<Int>)   // NRC values; others → failure
@Serializable public enum class CancellationPolicy { RUN_CLEANUP, RUN_CLEANUP_AND_STOP_ROUTINE }
```

Rules: `RetryPolicy.maxAttempts` must be `1` for plans of type `WRITE_VALUE`, `CLEAR_DTC`, `BASIC_SETTING`, `SERVICE_ROUTINE` (no automatic retry of mutating plans). For read plans `maxAttempts in 1..3`, `retryOn ⊆ {PROTOCOL_TIMEOUT, ADAPTER_NO_DATA, ADAPTER_BUS_ERROR}`.

```kotlin
@Serializable
public sealed interface ProtocolStep {
    val stepId: String

    @Serializable @SerialName("openChannel") public data class OpenChannel(override val stepId: String) : ProtocolStep
    @Serializable @SerialName("changeSession") public data class ChangeSession(override val stepId: String, val sessionType: Int) : ProtocolStep
    @Serializable @SerialName("securityAccess") public data class SecurityAccess(override val stepId: String, val level: Int) : ProtocolStep
    @Serializable @SerialName("request") public data class Request(
        override val stepId: String,
        val role: StepRole,
        val template: List<TemplatePart>,
        val expect: ResponseExpectation,
        val storeAs: String?,              // register name for the full positive response hex
    ) : ProtocolStep
    @Serializable @SerialName("decode") public data class Decode(override val stepId: String, val fromRegister: String, val codecId: String, val storeAs: String) : ProtocolStep
    @Serializable @SerialName("encode") public data class Encode(override val stepId: String, val baseRegister: String, val codecId: String, val inputRegister: String, val storeAs: String) : ProtocolStep
    @Serializable @SerialName("compare") public data class Compare(override val stepId: String, val leftRegister: String, val rightRegister: String, val mode: CompareMode) : ProtocolStep
    @Serializable @SerialName("pollUntil") public data class PollUntil(
        override val stepId: String, val request: Request, val statusCodecId: String,
        val successValues: List<Long>, val failureValues: List<Long>, val inProgressValues: List<Long>,
        val intervalMs: Long, val maxDurationMs: Long,
    ) : ProtocolStep
    @Serializable @SerialName("testerPresent") public data class TesterPresent(override val stepId: String, val enabled: Boolean) : ProtocolStep
    @Serializable @SerialName("delay") public data class Delay(override val stepId: String, val durationMs: Long) : ProtocolStep
    @Serializable @SerialName("assertPrecondition") public data class AssertPrecondition(override val stepId: String, val preconditionId: String) : ProtocolStep
}

@Serializable public enum class StepRole {
    GENERIC, IDENTIFY, READ_DTC, CLEAR_DTC, SELECT_ADAPTATION_CHANNEL, READ_CURRENT_VALUE, WRITE_VALUE, READ_BACK,
    START_ROUTINE, STOP_ROUTINE, POLL_ROUTINE, DISCOVERY,
}
@Serializable public enum class CompareMode { EXACT_BYTES, TARGET_FIELD_ONLY }

@Serializable
public sealed interface TemplatePart {
    @Serializable @SerialName("hex") public data class Literal(val hex: String) : TemplatePart
    @Serializable @SerialName("register") public data class FromRegister(val register: String, val sliceFrom: Int, val sliceLength: Int?) : TemplatePart
}

@Serializable
public data class ResponseExpectation(
    val positivePrefixHex: String,           // e.g. "62f1a3"
    val minTotalLength: Int,                 // bytes including prefix
    val exactTotalLength: Int?,
    val payloadLengthMultipleOf: Int?,       // applied to bytes after the prefix
)
```

Mapping of spec §7.7 step families:

| Spec family | Representation |
|---|---|
| CONNECT, SET_ADAPTER_MODE, OPEN_PROTOCOL_SESSION | `OpenChannel` (the executor derives adapter configuration from `ProtocolProfile.link`) |
| CHANGE_DIAGNOSTIC_SESSION | `ChangeSession` (UDS only) |
| SECURITY_REQUEST | `SecurityAccess` |
| SEND_REQUEST + EXPECT_RESPONSE | `Request` |
| PARSE_RESPONSE | `Decode` |
| SELECT_ADAPTATION_CHANNEL, READ_CURRENT_VALUE, WRITE_VALUE, READ_BACK, START_ROUTINE | `Request` with the matching `StepRole` |
| ENCODE_VALUE | `Encode` |
| POLL_ROUTINE | `PollUntil` |
| TESTER_PRESENT | `TesterPresent` |
| COMPARE | `Compare` |
| CLEANUP | `OperationPlan.cleanupSteps` |
| DELAY | `Delay` |
| ASSERT_PRECONDITION | `AssertPrecondition` |

Register names regex `^[a-z][a-zA-Z0-9_]{0,31}$`. Reserved registers written by the executor: `input` (value supplied by the caller, hex), `channel` (adaptation channel, hex). A step that reads an unset register fails with `AutoDiagError.Internal.InvariantViolation`; `RuntimePackValidator` statically rejects plans that read a register no earlier step writes.

## 10. Features and variants

```kotlin
@Serializable
public data class FeatureDefinition(
    val featureId: String,                   // e.g. "vag.car_setting_alarm"
    val canonicalKey: String,                // source key, e.g. "car_setting_alarm"
    val display: DisplayMetadata,
    val category: FeatureCategory,
    val interactionType: InteractionType,
    val variantIds: List<String>,            // spec "variants[]"; sorted ascending; resolved against RuntimePack.variants
    val evidenceStatus: EvidenceStatus,      // derived: EXACT iff at least one variant is EXACT
)

@Serializable
public data class DisplayMetadata(
    val titleKey: String,                    // Android string resource name "feature_<sanitized featureId>"
    val fallbackTitle: String,               // English, generated by the compiler from canonicalKey (never copied UI text)
    val descriptionKey: String?,
    val advisoryChecklistKeys: List<String>, // user checklist items (advisory, not safety gates)
)

@Serializable
public data class FeatureVariant(
    val variantId: String,                   // e.g. "vag.car_setting_alarm.v3"
    val featureId: String,
    val ecuLogicalAddress: Int,
    val platformSelector: List<PlatformFamily>, // empty = no platform constraint
    val applicability: ApplicabilityRule,
    val protocolProfileId: String,
    val valueTarget: ValueTarget,
    val readPlanId: String,
    val writePlanId: String?,                // null ⇒ read-only variant
    val codecId: String,
    val sessionRequirement: Int?,            // UDS session required for write; null = default session
    val securityRequirement: SecurityRequirement,
    val preconditionIds: List<String>,
    val postconditionIds: List<String>,
    val backupPolicy: BackupPolicy,
    val verificationPolicy: VerificationPolicy,
    val rollbackPolicy: RollbackPolicy,
    val evidenceIds: List<String>,
    val status: EvidenceStatus,
    val statusReasons: List<StatusReason>,   // empty iff status == EXACT
    val source: VariantSource,
)

@Serializable
public sealed interface ValueTarget {
    @Serializable @SerialName("udsDid") public data class UdsDid(val did: Int) : ValueTarget                 // value block = response bytes after "62 <did hi> <did lo>"
    @Serializable @SerialName("udsCoding") public data class UdsCoding(val did: Int) : ValueTarget           // same layout as UdsDid; whole block written back
    @Serializable @SerialName("tp20Adaptation") public data class Tp20Adaptation(val channel: Int) : ValueTarget // layout BLOCKED_EVIDENCE (EV-KWP-03)
    @Serializable @SerialName("tp20Coding") public data object Tp20Coding : ValueTarget                     // layout BLOCKED_EVIDENCE (EV-KWP-03)
    @Serializable @SerialName("submodule") public data class SubmoduleCoding(val parent: ValueTarget, val submoduleIndex: Int) : ValueTarget
}

@Serializable public sealed interface SecurityRequirement {
    @Serializable @SerialName("none") public data object None : SecurityRequirement
    @Serializable @SerialName("securityAccess") public data class SecurityAccess(val level: Int) : SecurityRequirement
    @Serializable @SerialName("sfd") public data object Sfd : SecurityRequirement
}
@Serializable public enum class BackupPolicy { RAW_VALUE_BLOCK }
@Serializable public enum class VerificationPolicy { READ_BACK_FULL_BLOCK }
@Serializable public enum class RollbackPolicy { NONE, REWRITE_ORIGINAL_BLOCK }

@Serializable
public data class StatusReason(val code: StatusReasonCode, val detail: String)
@Serializable public enum class StatusReasonCode {
    MISSING_INTERPRETATION, MISSING_TARGET, MISSING_OR_INVALID_MASK, MISSING_OR_INVALID_OFFSET, MISSING_WHITELIST,
    WHITELIST_CONTENT_BLOCKED, NOT_REPRODUCIBLE, HEURISTIC_PROVENANCE, PROTOCOL_CONFLICT, DID_CONFLICT,
    REJECTED_BUILDER, LAYOUT_BLOCKED, PROFILE_BLOCKED, SECURITY_REQUIRED, SFD_REQUIRED, UNRESOLVED_ECU_TRANSPORT,
    SOURCE_PARTIAL, UNKNOWN_CODEC, NO_READ_BACK,
}

@Serializable
public data class VariantSource(
    val settingKey: String,
    val sourceVariantId: String,             // e.g. "car_setting_ads_step_1_var_1"
    val concreteClass: String,
    val ecuSymbol: String,
    val whitelistSymbol: String?,
    val didOrChannelHex: String?,
    val asmAddress: String,
    val sourceVariantStatus: String,         // "EXACT_VERIFIED" | "PARTIAL"
    val reproducible: Boolean,
    val provenanceMethod: String,            // "CFG_DATAFLOW" | "HEURISTIC_SLICE" | "UNKNOWN"
)
```

Validation: `status == EXACT` requires every rule in `EXECUTABLE_VARIANT_RULES` (`RUNTIME_PACK_SPEC.md` §7). `writePlanId != null && rollbackPolicy == REWRITE_ORIGINAL_BLOCK` requires the write plan to accept any value block of the original length.

## 11. Preconditions

```kotlin
@Serializable
public data class PreconditionDefinition(
    val preconditionId: String,
    val kind: PreconditionKind,
    val minMillivolts: Int?,                 // for MIN_VOLTAGE
    val labelKey: String,
    val evidenceIds: List<String>,
    val status: EvidenceStatus,
)
@Serializable public enum class PreconditionKind { MIN_VOLTAGE, USER_CONFIRMATION }
```

Only these two kinds are machine-checkable in v1. Every automotive precondition without an evidenced machine check (ignition on, engine off, level ground) is shown as an advisory checklist item that the user must confirm (`USER_CONFIRMATION`).

## 12. Live data

```kotlin
@Serializable
public data class LiveDataParameter(
    val parameterId: String,
    val ecuLogicalAddress: Int,
    val protocolProfileId: String,
    val requestPlanId: String,               // plan of type READ_LIVE_DATA
    val responseSelector: ResponseSelector,
    val codecId: String,                     // ScaledInt, UnsignedInt, SignedInt, BooleanBit, EnumMask or Ascii
    val pollGroup: String,                   // parameters with equal pollGroup share one request
    val minimumPollIntervalMs: Long,
    val evidenceIds: List<String>,           // spec "parserEvidence"
    val status: EvidenceStatus,
)
@Serializable
public data class ResponseSelector(val positivePrefixHex: String, val valueBlockOffset: Int)
```

Spec §21 fields `offset/length/signedness/endianness/scale/additiveOffset/unit/validRange` live in the referenced `ValueCodec`.

## 13. Runtime pack types

```kotlin
// package com.guns96x.autodiag.core.model.pack
@Serializable
public data class RuntimePackManifest(
    val schemaVersion: Int,
    val packId: String,                      // "vag"
    val packVersion: String,                 // semver "MAJOR.MINOR.PATCH"
    val buildCommit: String,                 // autodiag-android commit (40 hex)
    val sourceCommit: String,                // ghidracarista commit (40 hex)
    val sourceArtifactHashes: Map<String, String>, // source path → SHA-256 hex
    val generatedAt: Long,
    val featureCount: Int,                   // must equal payload.features.size
    val variantCount: Int,                   // must equal payload.variants.size
    val evidenceSummary: Map<EvidenceStatus, Int>, // counts over variants; must equal recomputation
    val payloadHash: String,                 // SHA-256 hex of payload.json bytes
    val minAppSchemaVersion: Int,
)

@Serializable
public data class RuntimePackPayload(
    val oem: OemDescriptor,
    val evidence: List<EvidenceRef>,
    val ecus: List<EcuDefinition>,
    val ecuVariants: List<EcuVariant>,
    val whitelists: List<WhitelistDefinition>,
    val protocolProfiles: List<ProtocolProfile>,
    val operationPlans: List<OperationPlan>,
    val codecs: List<ValueCodec>,
    val preconditions: List<PreconditionDefinition>,
    val features: List<FeatureDefinition>,
    val variants: List<FeatureVariant>,
    val liveDataParameters: List<LiveDataParameter>,
    val serviceProcedures: List<ServiceProcedure>,
    val diagnosticOperations: List<DiagnosticOperation>,
    val platformRules: List<PlatformRule>,
)

@Serializable public data class OemDescriptor(val manufacturer: Manufacturer, val oemModule: String) // "oem-vag"
@Serializable public data class PlatformRule(val platform: PlatformFamily, val rule: ApplicabilityRule, val evidenceIds: List<String>, val status: EvidenceStatus)
@Serializable public data class DiagnosticOperation(val operationId: String, val kind: OperationType, val protocolProfileId: String, val operationPlanId: String, val evidenceIds: List<String>, val status: EvidenceStatus)

@Serializable
public data class ServiceProcedure(
    val procedureId: String,                 // e.g. "vag.tool_epb.open"
    val kind: ServiceProcedureKind,
    val ecuLogicalAddress: Int,
    val operationPlanId: String,
    val preconditionIds: List<String>,
    val display: DisplayMetadata,
    val evidenceIds: List<String>,
    val status: EvidenceStatus,
)
@Serializable public enum class ServiceProcedureKind { BASIC_SETTING, SERVICE_ROUTINE }

public data class RuntimePack(val manifest: RuntimePackManifest, val payload: RuntimePackPayload, val index: RuntimePackIndex)
// RuntimePackIndex (not serialized): maps id → entity for every list, built once by RuntimePackLoader.
```

## 14. Write transactions and backups

```kotlin
// package com.guns96x.autodiag.core.model.write
@Serializable
public data class WriteTransaction(
    val transactionId: String,               // UUID v4
    val vehicleId: String,
    val ecuLogicalAddress: Int,
    val featureId: String,
    val variantId: String,
    val runtimePackVersion: String,
    val requestedValue: DecodedValue,
    val state: WriteTransactionState,        // STATE_MACHINES.md §9
    val originalBlockHex: String?,
    val encodedBlockHex: String?,
    val readBackBlockHex: String?,
    val backupId: String?,
    val failureCode: String?,                // AutoDiagError.code
    val createdAt: Long,
    val updatedAt: Long,
    val stateHistory: List<WriteStateEntry>,
)
@Serializable public data class WriteStateEntry(val state: WriteTransactionState, val at: Long, val errorCode: String?)
@Serializable public enum class WriteTransactionState {
    CREATED, PRECHECKING, READING_ORIGINAL, BACKING_UP, ENCODING, WRITING, VERIFYING, COMMITTED,
    ABORTED_BEFORE_WRITE, REJECTED_BY_ECU, INTERRUPTED_UNKNOWN_STATE, VERIFICATION_MISMATCH,
    ROLLING_BACK, ROLLED_BACK, ROLLBACK_FAILED,
}

@Serializable
public data class BackupRecord(
    val backupId: String,                    // UUID v4
    val vehicleId: String,
    val ecuIdentitySnapshot: EcuIdentity,
    val featureId: String,
    val variantId: String,
    val rawBeforeHex: String,
    val decodedBefore: DecodedValue,
    val runtimePackVersion: String,
    val timestamp: Long,
    val evidenceHash: String,                // SHA-256 hex over canonical JSON of the variant's EvidenceRefs sorted by id
)
```

## 15. Traces

```kotlin
// core-common, package com.guns96x.autodiag.core.common.trace
public data class TraceEvent(
    val sessionId: String,
    val sequence: Long,                      // strictly increasing per session, starts at 0
    val timestamp: Long,                     // epoch ms
    val monotonicNanos: Long,
    val layer: TraceLayer,                   // TRANSPORT, ADAPTER, CAN, ISOTP, UDS, KWP, TP20, EXECUTOR
    val direction: TraceDirection,           // TX, RX, INFO
    val transport: String?,                  // TransportKind name
    val adapter: String?,                    // AdapterFamily name
    val protocol: ProtocolFamily?,
    val canTxId: Int?,
    val canRxId: Int?,
    val payloadHex: String,
    val elapsedMs: Long?,                    // RX events: time since the matching TX
    val nrc: Int?,
    val decodedOperation: String?,           // e.g. "UDS ReadDataByIdentifier F187"
    val sessionState: String?,
    val correlation: TraceCorrelation?,
)
public data class TraceCorrelation(
    val operationId: String?, val featureId: String?, val variantId: String?, val ecuLogicalAddress: Int?,
    val evidenceIds: List<String>, val transactionId: String?,
)
```

`TraceEvent` is serialized by `:storage` and `:tools:trace-toolchain` through a `@Serializable` mirror `TraceEventDto` in `core-model` (package `core.model.trace`) with identical fields; `core-common` stays serialization-free.

## 16. Diagnostics results

```kotlin
// package com.guns96x.autodiag.core.model.diagnostics
@Serializable
public data class ScanReport(
    val scanId: String, val vehicleId: String, val startedAt: Long, val finishedAt: Long?,
    val ecuResults: List<EcuScanResult>, val runtimePackVersion: String,
)
@Serializable
public data class EcuScanResult(
    val logicalAddress: Int, val identity: EcuIdentity?, val listedInGateway: Boolean,
    val status: EcuScanStatus, val dtcs: List<DtcRecord>, val errorCode: String?,
)
@Serializable public enum class EcuScanStatus { OK_NO_FAULTS, OK_WITH_FAULTS, NOT_RESPONDING, NOT_SUPPORTED, ERROR, CANCELLED }
@Serializable
public data class DtcRecord(
    val codeHex: String,                     // 3 bytes (UDS) or 2 bytes (KWP)
    val displayCode: String,                 // UDS: SAE J2012 format P/C/B/U + 4 hex digits from the first 2 bytes, then "-" + third byte hex
    val statusByte: Int,
    val statusFlags: List<DtcStatusFlag>,
    val protocol: ProtocolFamily,
)
@Serializable public enum class DtcStatusFlag {
    TEST_FAILED, TEST_FAILED_THIS_CYCLE, PENDING, CONFIRMED, NOT_COMPLETED_SINCE_CLEAR, FAILED_SINCE_CLEAR,
    NOT_COMPLETED_THIS_CYCLE, WARNING_INDICATOR,
}
```

UDS status-byte bits follow ISO 14229-1 (bit 0 … bit 7 in the enum order above). KWP status-byte meaning is BLOCKED_EVIDENCE: for KWP records `statusFlags` is empty and only `statusByte` is shown.

## 17. Master-spec field mapping

| Spec | Contract |
|---|---|
| §7.1 `gatewayIdentity` | `gatewayLogicalAddress` + gateway `EcuIdentity` |
| §7.3 predicate/rule fields | `EcuVariant.predicate` |
| §7.4 `variants[]` | `FeatureDefinition.variantIds` |
| §7.5 `ecuSelector` | `ecuLogicalAddress` |
| §7.5 `readPlan`/`writePlan`/`valueCodec` | `readPlanId`/`writePlanId`/`codecId` |
| §7.5 `preconditions[]`/`postconditions[]` | `preconditionIds`/`postconditionIds` |
| §8 `confidence` | `EvidenceConfidence` |
| §21 `parserEvidence` | `LiveDataParameter.evidenceIds` |
| §27 backup fields | `BackupRecord` (all fields present) |
| §28 trace fields | `TraceEvent` (`adapterCommand` = ADAPTER-layer TX `payloadHex`; `txId/rxId` = `canTxId/canRxId`; `elapsedTime` = `elapsedMs`; `NRC` = `nrc`; `evidenceRef` = `correlation.evidenceIds`) |
