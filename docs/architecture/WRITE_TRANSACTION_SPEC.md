# Write Transaction Specification

**Status:** NORMATIVE.

## 1. Entry requirements

Write is rejected before transport activity unless:
- variant selection result is `ExactMatch`;
- variant status is EXACT;
- write plan exists and is EXACT;
- codec exists and is EXACT;
- protocol consistency passes;
- ECU identity snapshot still matches the variant applicability;
- adapter capabilities satisfy the profile;
- security requirement is None or a legitimate configured provider is available.

## 2. Exclusive lock

One `Mutex` per vehicle session. Write and service operations acquire it. Live-data polling pauses before lock grant.

## 3. Sequence

1. persist transaction in CREATED;
2. PRECHECKING: re-evaluate identity/applicability/capabilities;
3. evaluate machine-checkable preconditions;
4. require user-confirmed advisory preconditions already present in `WriteRequest.confirmedPreconditionIds`;
5. READING_ORIGINAL: execute read plan;
6. BACKING_UP: persist immutable `BackupRecord`;
7. ENCODING: apply codec only to target field; validate untouched bytes unchanged;
8. WRITING: execute exact write plan;
9. VERIFYING: execute read plan again;
10. compare by `VerificationPolicy`;
11. COMMITTED only after success criterion passes.

## 4. Voltage

If adapter capability exposes voltage and variant includes a MIN_VOLTAGE precondition, a value below threshold blocks before write. If voltage is required by the plan but cannot be measured, result is `PreconditionFailed`.

## 5. Retry

Automatically retryable before write:
- transport connect timeout;
- read-only identity/read timeout;
- NRC 0x78 according to protocol policy.

Never automatically retried after write transmission:
- write request timeout;
- disconnect;
- unknown response;
- read-back mismatch;
- service start request.

## 6. Disconnect

- before WRITING: `ABORTED_BEFORE_WRITE`.
- while/after WRITING before verified response state is known: `INTERRUPTED_UNKNOWN_STATE`.
- reconnect may be offered only to perform identity/read-back recovery. The write is not retransmitted automatically.

## 7. Read-back mismatch

Transition to `VERIFICATION_MISMATCH`; persist expected and observed blocks. Rollback is considered only by §8.

## 8. Rollback

Allowed only when:
- rollbackPolicy == REWRITE_ORIGINAL_BLOCK;
- exact original raw block exists;
- same ECU identity is revalidated;
- reverse write uses the same exact evidenced write plan;
- protocol session can be safely re-established.

Rollback itself is a write transaction subflow and must be read-back verified.

## 9. Identity change

Any identity field used by the selected applicability rule changing between precheck and write causes `EcuIdentityChanged` and aborts before transmission.

## 10. Persistence

Every state transition is appended to `stateHistory` and persisted before the next external side effect.
