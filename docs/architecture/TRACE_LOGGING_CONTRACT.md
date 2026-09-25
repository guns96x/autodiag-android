# Trace and Logging Contract

**Status:** NORMATIVE.

## Structured trace

The canonical trace event is `TraceEvent` in `DATA_MODEL_CONTRACT.md`.

Mandatory:
- sessionId
- sequence
- timestamp
- monotonicNanos
- layer
- direction
- payloadHex

Context fields are filled when known:
- transport
- adapter
- protocol
- CAN TX/RX IDs
- elapsedMs
- NRC
- decodedOperation
- sessionState
- operationId
- featureId
- variantId
- ECU logical address
- evidence IDs
- transactionId

Sequence is strictly increasing per session.

## Layers

TRANSPORT, ADAPTER, CAN, ISOTP, UDS, KWP, TP20, EXECUTOR.

## Production logger

Categories: TRANSPORT, ADAPTER, CAN, ISOTP, UDS, KWP, TP20, DISCOVERY, IDENTITY, APPLICABILITY, FEATURE, SERVICE, SAFETY, STORAGE, EVIDENCE.

Production logs do not include VIN or full raw identity payloads. Engineering trace storage may include raw diagnostic payloads because it is local and user-controlled.

## Persistence

Trace events are buffered in bounded memory batches and persisted asynchronously. On buffer pressure, INFO events may be dropped first; TX/RX events for active writes are never dropped.

## Export

Trace export is user initiated and includes runtime pack version/source commit plus a SHA-256 hash of the exported event stream.
