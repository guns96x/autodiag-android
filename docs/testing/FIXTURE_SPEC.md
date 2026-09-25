# Fixture Specification

**Status:** NORMATIVE.

Fixtures contain recorded or hand-authored evidence-backed request/response pairs plus metadata. Synthetic failure fixtures may mutate timing/framing but never invent successful automotive semantics.

## Directory

```
testkit/src/testFixtures/
  pq35_golf5_bls/
  mqb_generic/
  meb_generic/
  failures/
```

Each vehicle fixture contains:
`fixture.json`, `adapter-transcript.jsonl`, `diagnostic-trace.jsonl`, `expected.json`.

## PQ35_Golf5_BLS

Target:
- VW Golf 5
- 1.9 TDI BLS
- PQ35
- EDC16U34

Until TP2 channel-setup evidence is resolved, this fixture may contain read-only identity/meta expectations but must mark TP2 executable replay `BLOCKED_EVIDENCE`. A captured trace can unblock it when committed with hash/provenance.

## MQB_GENERIC / MEB_GENERIC

Only evidence-backed operations are present. Generic means platform-shaped fixture, not permission to invent ECU behavior. Missing operations are omitted.

## Failure fixtures

Required deterministic fixtures:
- adapter_disconnect_before_write
- adapter_disconnect_after_write
- transport_timeout
- uds_negative_response
- uds_response_pending_then_success
- uds_response_pending_timeout
- low_voltage
- readback_mismatch
- ambiguous_variant
- corrupted_runtime_pack
- identity_changed
- malformed_can_frame
- isotp_sequence_error
- tp20_ack_timeout

Each fixture declares `expectedErrorCode` and expected terminal state.

## Provenance

Successful automotive response fixtures include evidence IDs or trace hash/source. Failure mutation metadata states which valid fixture was transformed and how.
