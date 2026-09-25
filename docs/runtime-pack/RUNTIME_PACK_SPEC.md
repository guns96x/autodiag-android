# Runtime Pack Specification

**Status:** NORMATIVE.

## 1. Files

A runtime pack directory contains exactly:

```
manifest.json
payload.json
signature.der
```

No executable code, scripts or class files are allowed.

## 2. Canonical serialization

- UTF-8.
- JSON object keys sorted lexicographically.
- arrays preserve semantic order only where order is meaningful; ID-indexed collections are sorted by ID before output.
- no insignificant whitespace in hashed/signed form.
- hex strings are lowercase, no separators.
- timestamps are epoch milliseconds UTC.

## 3. Manifest

Fields exactly match `RuntimePackManifest` in `DATA_MODEL_CONTRACT.md`.

`payloadHash` = SHA-256 of canonical `payload.json` bytes.

`signature.der` is ECDSA P-256/SHA-256 over canonical `manifest.json` bytes after `payloadHash` is populated.

Debug/test packs use the committed test public key. Release packs use the release public key injected through BuildConfig.

## 4. Payload collections

`RuntimePackPayload` contains:
- oem
- evidence
- ecus
- ecuVariants
- whitelists
- protocolProfiles
- operationPlans
- codecs
- preconditions
- features
- variants
- liveDataParameters
- serviceProcedures
- diagnosticOperations
- platformRules

Every referenced ID must resolve exactly once.

## 5. Activation

Validation order:
1. JSON parse;
2. schema version compatibility;
3. payload hash;
4. signature;
5. ID uniqueness;
6. reference integrity;
7. model validation;
8. evidence rules;
9. protocol consistency;
10. executable-variant rules;
11. manifest count recomputation.

Any failure rejects the pack. Activation writes the new pack to a temporary location, fsyncs, updates Room metadata in one transaction, then atomically swaps current pointer. Last valid pack remains available.

## 6. Evidence rules

EXACT executable entity requires:
- all referenced EvidenceRefs VERIFIED or OFFICIAL_PROTOCOL;
- source-backed evidence has sourceCommit/sourceFile/artifactHash;
- every requested command/response parser field has at least one evidence reference;
- no status reason;
- no BLOCKED_EVIDENCE register dependency.

## 7. Executable variant rules

A variant may be executable for read iff:
- status EXACT;
- readPlanId resolves to EXACT plan;
- codecId resolves to EXACT codec;
- protocol consistency passes;
- applicability contains no blocked whitelist/rule.

Writable additionally requires:
- writePlanId non-null and EXACT;
- read-back verification policy;
- backup policy;
- exact value target;
- reproducible source = true;
- provenanceMethod = CFG_DATAFLOW or equivalent explicit exact provenance;
- no security requirement lacking legitimate provider.

## 8. Compiler downgrade rules

Missing field -> downgrade, never default.

- missing applicability evidence -> PARTIAL
- protocol disagreement -> CONFLICT
- security/protection without provider -> PROTECTED
- non-reproducible extraction -> PARTIAL
- heuristic provenance -> PARTIAL
- source explicitly rejected by evidence register -> UNRESOLVED or CONFLICT according to register
- missing read-back for write -> PARTIAL

## 9. ghidracarista import

The compiler pins one source commit. It may import only files listed in its source manifest.

Generated/known-problem files rejected by `EVIDENCE_REGISTER.md` must not become executable.

A future `REPRODUCIBILITY_ATTESTATION.json` may upgrade source variants only when it contains per-variant:
- sourceVariantId
- binary artifact SHA-256
- exact factory/call target
- exact argument provenance method
- extraction tool commit/version
- reproducibility status

Absent attestation means `reproducible=false`.

## 10. Rollback

Keep at least current and previous valid pack. Activation failure never deletes either. User/admin may explicitly reactivate the previous valid pack.

## 11. Minimal valid pack

See `docs/runtime-pack/examples/minimal-readonly-pack.json`. The example is read-only and intentionally contains no write variant.
