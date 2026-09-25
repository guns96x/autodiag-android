# Protocol Consistency

**Status:** NORMATIVE.

A variant is executable only when these four facts agree:

1. `FeatureVariant.valueTarget`
2. `FeatureVariant.protocolProfileId`
3. resolved ECU `transportAddress` / `protocolFamily`
4. every `OperationPlan.protocolProfileId` and request service in its read/write plans

## Rules

### UDS
- `ValueTarget.UdsDid` and `ValueTarget.UdsCoding` require an ISO-TP-backed UDS profile.
- ECU `protocolFamily == UDS`.
- request plans may use UDS services only.
- a UDS service may never execute over a TP2 profile.

### TP2/KWP
- `ValueTarget.Tp20Adaptation`, `Tp20Coding`, and TP2 submodule coding require a TP2 link carrying KWP2000 payloads.
- ECU `protocolFamily == KWP2000`.
- request plan uses evidenced KWP request prefixes.
- TP2 adaptation flow is select channel -> read -> write -> read-back; no generic UDS substitution.

### OBD2
- OBD2 operation plans are read-only in v1 unless a separately evidenced write profile is introduced by ADR.

## Validator output

Any mismatch yields `StatusReasonCode.PROTOCOL_CONFLICT` and effective status `CONFLICT`.

## No inference rule

The compiler may not reinterpret a source class name to make a protocol conflict disappear. The evidence must be corrected upstream or the variant remains blocked.
