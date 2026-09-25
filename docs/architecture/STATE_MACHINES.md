# State Machines

**Status:** NORMATIVE.

## 1. Connection

`IDLE -> CONNECTING -> CONNECTED -> INITIALIZING_ADAPTER -> READY`

Failures from CONNECTING/INITIALIZING -> `FAILED`.  
Remote close from READY -> `DISCONNECTED`.  
Explicit user stop -> `CLOSED`.  
Reconnect is a new transition `DISCONNECTED -> CONNECTING`, never an implicit continuation of a write.

## 2. Adapter initialization

`NEW -> RESETTING -> BASELINE_CONFIG -> PROBING -> CONFIGURED`

Any unsupported probe marks capability state and continues. Fatal transport loss -> FAILED.

## 3. ISO-TP

RX:
`IDLE -> SINGLE_COMPLETE` or `FIRST_FRAME -> RECEIVING_CONSECUTIVE -> COMPLETE`.  
Sequence mismatch, timeout, invalid length -> FAILED.

TX:
`IDLE -> SINGLE_SENT` or `FIRST_SENT -> WAIT_FC -> SEND_CF -> COMPLETE`.  
FC WAIT is bounded by profile policy. FC OVERFLOW -> FAILED.

## 4. UDS session

`DEFAULT -> ENTERING_EXTENDED -> EXTENDED`.  
Tester-present runs only while a non-default session is held and the channel remains open.  
NRC 0x78 keeps request in `WAITING_PENDING` until P2* timeout.  
Channel close -> DEFAULT/closed.

## 5. TP2 session

`CLOSED -> CHANNEL_SETUP -> PARAMETER_EXCHANGE -> OPEN -> CLOSING -> CLOSED`.

If channel-setup/addressing evidence is blocked, runtime returns `EvidenceError.Blocked` before transmission. ACK/block-sequence failure -> FAILED -> CLOSED.

## 6. Vehicle discovery

`CONNECT -> PROBE_ADAPTER -> PROBE_PROTOCOLS -> DISCOVER_GATEWAY -> ENUMERATE_ECUS -> IDENTIFY_ECUS -> RESOLVE_PLATFORM -> COMPLETE`.

Partial ECU identity failure does not abort the entire report; the ECU becomes unresolved/not responding.

## 7. AutoScan

`PREPARE -> DISCOVER/LOAD_ECUS -> FOR_EACH_ECU_IDENTITY -> READ_DTC -> PERSIST -> COMPLETE`.

Per-ECU failure is isolated. User cancellation produces a valid partial report with `CANCELLED` statuses.

## 8. Live-data session

`IDLE -> STARTING -> POLLING -> PAUSED_EXCLUSIVE_OPERATION -> POLLING -> STOPPING -> IDLE`.

Backpressure drops superseded samples, never request/response ordering.

## 9. Write transaction

States are exactly `WriteTransactionState` from `DATA_MODEL_CONTRACT.md`.

Normal path:
`CREATED -> PRECHECKING -> READING_ORIGINAL -> BACKING_UP -> ENCODING -> WRITING -> VERIFYING -> COMMITTED`.

Before first write, any failure -> `ABORTED_BEFORE_WRITE`.  
ECU negative write response -> `REJECTED_BY_ECU`.  
Transport uncertainty after write transmit -> `INTERRUPTED_UNKNOWN_STATE`.  
Read-back mismatch -> `VERIFICATION_MISMATCH`.  
Rollback path: `ROLLING_BACK -> ROLLED_BACK | ROLLBACK_FAILED`.

## 10. Service routine

`PRECHECKING -> STARTING -> RUNNING -> POLLING -> SUCCEEDED | FAILED | CANCEL_REQUESTED -> CLEANUP -> FINISHED`.

Cancellation invokes explicit stop step only when the operation plan contains one. Cleanup always runs in `NonCancellable`.
