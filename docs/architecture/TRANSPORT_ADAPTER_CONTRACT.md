# Transport and Adapter Contract

**Status:** NORMATIVE.

## 1. Transport separation

Transport owns bytes and connection lifecycle only. Adapter owns prompt/AT/ST command dialect. Protocol modules own automotive framing.

## 2. Bluetooth Classic SPP

- RFCOMM UUID: `00001101-0000-1000-8000-00805F9B34FB`.
- connect timeout: 10_000 ms.
- automatic connect attempts: initial + 2 retries, with 500 ms then 1_000 ms delay.
- remote disconnect transitions to `TransportState.Disconnected(RemoteClosed)`.
- no write is automatically replayed after reconnect.

## 3. BLE

Profiles are explicit data records:

```kotlin
public data class BleProfile(
    val serviceUuid: UUID,
    val writeCharacteristicUuid: UUID,
    val notifyCharacteristicUuid: UUID,
    val cccdUuid: UUID = UUID.fromString("00002902-0000-1000-8000-00805f9b34fb"),
)
```

- request MTU 512; use negotiated result.
- ATT payload chunk size = `max(1, negotiatedMtu - 3)`.
- one GATT write at a time.
- profile with missing evidenced UUIDs is unavailable, not guessed.

## 4. USB

USB Host supports only device/interface classes explicitly matched by the implementation. FTDI and CDC-ACM paths are separate drivers. Unknown device profile -> `AdapterError.UnsupportedDevice`.

## 5. Adapter capabilities

Each capability state is:

```kotlin
public enum class CapabilityState { SUPPORTED, UNSUPPORTED, UNKNOWN, BROKEN_IMPLEMENTATION }
```

Capability keys include:
- CAN_11BIT
- CAN_29BIT
- ISO_TP_OFFLOAD
- RAW_CAN
- CUSTOM_HEADER
- CUSTOM_FILTER
- EXTENDED_ADDRESSING
- TP20
- K_LINE

Product/device name is a sorting hint only and cannot mark a capability SUPPORTED.

## 6. ELM/STN command policy

- command delimiter: CR.
- prompt: `>`.
- standard command timeout: 2_000 ms.
- diagnostic bus-init timeout: 5_000 ms.
- normal diagnostic response timeout: 5_000 ms.
- extended routine timeout: operation-plan specific; absent evidence makes the plan non-executable.
- unsupported response `?` downgrades the probed capability.
- `NO DATA`, `CAN ERROR`, `BUS BUSY`, `FB ERROR`, `BUFFER FULL`, `ERRxx` map to canonical AdapterError/ProtocolError values.

## 7. Initialization baseline

Evidence-backed baseline commands:
`ATZ`, `ATE0`, `ATL0`, `ATS0`, `ATH1`, `ATAT1`.

Protocol-specific commands are selected only from the runtime profile and evidence register.

## 8. Reconnect

Read-only sessions may reconnect once after reinitialization and ECU re-identification. Active write/service transactions follow `WRITE_TRANSACTION_SPEC.md` and are never resumed blindly.
