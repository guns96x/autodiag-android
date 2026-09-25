# UI Contract

**Status:** NORMATIVE.

## Navigation

Top-level routes:
- Garage
- Vehicle Overview
- Health/Scan
- ECUs
- Faults
- Live Data
- Customizations
- Service
- History
- Connection
- Settings
- Developer/Traces (debug/engineering builds; read-only inspection in release if enabled)

## Ownership

Each screen has one ViewModel. ViewModels depend on use cases only.

## Screen state pattern

Every ViewModel exposes one `StateFlow<ScreenState>` and accepts typed `UiAction`.

Shared state:
```kotlin
public sealed interface AsyncUiState<out T> {
    public data object Idle : AsyncUiState<Nothing>
    public data object Loading : AsyncUiState<Nothing>
    public data class Data<T>(val value: T) : AsyncUiState<T>
    public data class Empty(val reason: String) : AsyncUiState<Nothing>
    public data class Error(val code: String, val recoverable: Boolean) : AsyncUiState<Nothing>
}
```

## Feature controls

Interaction mapping:
- TOGGLE -> switch with explicit current/new value
- ENUM_SELECTION -> single-choice list
- NUMERIC_RANGE -> slider + numeric text
- NUMERIC_INPUT -> validated numeric field
- ACTION -> confirmation action
- WIZARD -> ordered pages from operation metadata
- SERVICE_ROUTINE -> prerequisite + progress + terminal result
- READ_ONLY -> value card

Unavailable states are explicit: UNSUPPORTED, PROTECTED, AMBIGUOUS, BLOCKED_EVIDENCE, ADAPTER_CAPABILITY_MISSING.

## Write UX

Before execution show:
- ECU name/address
- feature
- current decoded value
- requested value
- runtime pack version
- backup policy
- required confirmations

Progress mirrors transaction state. Success appears only on COMMITTED/ROLLED_BACK where applicable. Interrupted/mismatch states show recovery data and never generic success.

## Accessibility

All actionable controls have semantic labels; status is not color-only; dynamic font scaling must not hide confirmation controls.
