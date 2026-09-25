# CAN, ISO-TP, UDS, KWP2000 and TP2.0 Implementation Plan

> **For agentic workers:** Execute only after transport/adapter gate.

**Goal:** Implement reusable protocol state machines with golden vectors, normalized errors and trace events, without OEM-specific feature logic.

**Architecture:** CAN frames feed ISO-TP; UDS/KWP consume payload channels; TP2.0 provides a KWP transport session. OEM modules configure addressing but do not modify protocol algorithms.

**Tech Stack:** Kotlin coroutines/Flow, deterministic clock/test scheduler, golden vector fixtures.

**Spec:** `docs/superpowers/specs/2026-09-25-autodiag-android-master-design.md`

## Global Constraints
- Protocol modules contain no OEM feature IDs, vehicle model assumptions, or UI logic.
- All timing/retry behavior is deterministic and testable with a fake clock.
- Unknown NRC/status values are preserved raw.
- SecurityAccess remains an interface; no unauthorized key derivation or bypass logic.
- Every evidenced request shape must have a golden-vector test before use by higher layers.


## Review Focus
- ISO-TP sequence mismatch;
- UDS NRC 0x78 response pending;
- session timeout/tester present;
- TP2 lost ACK/block;
- cancellation during segmented transfer.

---

### Task 1: CAN frame model and trace events

**Files:**
- `protocol-can/.../CanFrame.kt`
- `protocol-can/.../CanAddress.kt`
- `protocol-can/.../CanChannel.kt`
- Tests for 11-bit/29-bit validation.

- [ ] Add tests rejecting out-of-range IDs and oversized frames for configured frame type.
- [ ] Implement model/channel abstraction.
- [ ] Emit structured trace events.
- [ ] Commit `feat(can): add CAN frame and channel model`.

### Task 2: ISO-TP

**Files:**
- `protocol-isotp/.../IsoTpSession.kt`
- `.../IsoTpEncoder.kt`
- `.../IsoTpDecoder.kt`
- `.../IsoTpTiming.kt`
- golden tests.

- [ ] Add vectors for SF, FF/CF/FC, BS, STmin, sequence wrap, timeout, malformed FC.
- [ ] Implement segmentation/reassembly and flow control.
- [ ] Use deterministic clock in tests.
- [ ] Commit `feat(isotp): implement ISO-TP state machine`.

### Task 3: UDS core

**Files:**
- `protocol-uds/.../UdsSession.kt`
- `.../UdsRequest.kt`
- `.../UdsResponse.kt`
- `.../UdsNrc.kt`
- `.../UdsServices.kt`

- [ ] Add golden vectors for 0x10, 0x22, 0x2E, 0x14, 0x19, 0x31, 0x3E.
- [ ] Add NRC tests including 0x78 response pending and unknown NRC retention.
- [ ] Implement tester-present scheduler and session lifecycle.
- [ ] Ensure security access is an interface only; no bypass algorithm.
- [ ] Commit `feat(uds): implement UDS session and core services`.

### Task 4: KWP2000 payload engine

**Files:**
- `protocol-kwp/.../KwpSession.kt`
- `.../KwpRequest.kt`
- `.../KwpResponse.kt`
- `.../KwpNegativeResponse.kt`

- [ ] Add golden vectors for evidenced local-identifier and routine-style payloads.
- [ ] Implement timing, response parsing and normalized errors.
- [ ] Commit `feat(kwp): implement KWP2000 payload engine`.

### Task 5: TP2.0 transport session

**Files:**
- `protocol-tp20/.../Tp20Session.kt`
- `.../Tp20ChannelSetup.kt`
- `.../Tp20BlockCodec.kt`
- `.../Tp20Timing.kt`

- [ ] Add vectors for channel setup, block sequence, ACK, retry, teardown and timeout.
- [ ] Implement KWP payload carriage.
- [ ] Add loss/duplicate/delayed-frame fault tests.
- [ ] Commit `feat(tp20): implement TP2.0 session transport`.

### Task 6: Cross-protocol request executor

**Files:**
- `core-runtime/.../DiagnosticChannel.kt`
- `core-runtime/.../DiagnosticExecutor.kt`
- `core-runtime/.../NormalizedDiagnosticError.kt`

- [ ] Test one fake ECU over UDS/ISO-TP and one over KWP/TP2 using the same executor contract.
- [ ] Implement cancellation and trace propagation.
- [ ] Commit `feat(runtime): add normalized diagnostic executor`.

### Gate
All golden vectors and fault-injection tests pass; no VAG feature names or Carista feature IDs exist in protocol modules.
