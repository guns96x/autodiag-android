# Storage Schema

**Status:** NORMATIVE.

Room database name: `autodiag.db`.

## Entities

### vehicles
PK `vehicleId TEXT`; VIN nullable unique index when non-null; manufacturer; platform; firstSeenAt; lastSeenAt.

### ecus
Composite PK `vehicleId, logicalAddress, identityHash`; identity JSON snapshot; firstSeenAt; lastSeenAt. Index `vehicleId, logicalAddress`.

### scan_sessions
PK `scanId`; FK vehicleId; startedAt; finishedAt; runtimePackVersion. Index vehicleId+startedAt.

### dtcs
Composite PK `scanId, logicalAddress, codeHex`; statusByte; protocol; displayCode.

### live_data_sessions
PK sessionId; FK vehicleId; startedAt; finishedAt.

### live_data_samples
Composite PK `sessionId, parameterId, timestamp, sequence`; encoded `DecodedValue` JSON. Index sessionId+timestamp.

### feature_backups
PK backupId; immutable; FK vehicleId; logicalAddress; featureId; variantId; rawBeforeHex; decodedBeforeJson; runtimePackVersion; evidenceHash; timestamp.

### write_transactions
PK transactionId; immutable terminal record plus append-only state-history child table `write_transaction_states`.

### operation_history
PK operationId; vehicleId; kind; feature/procedure nullable; resultCode; startedAt; finishedAt.

### diagnostic_traces
Composite PK sessionId+sequence; all TraceEventDto fields. Index sessionId+sequence.

### runtime_packs
PK packId+packVersion; manifest JSON; payload path/hash; sourceCommit; installedAt; active boolean. Unique partial policy enforced in DAO transaction: one active version per packId.

### connection_profiles
PK profileId; transport kind; device identifier; adapter family; lastKnownCapabilitiesJson; updatedAt.

## Retention

- backups/write transactions: never auto-delete;
- scan history: never auto-delete in v1;
- live-data samples/traces: user may delete per session;
- runtime packs: current + previous valid version retained; older versions user-removable after no backup references depend on them.

## Migrations

Every schema version N->N+1 has an explicit Room Migration and a test from each supported historical schema. Destructive migration is forbidden.

## Export

User-initiated exports use JSON/CSV as appropriate and never mutate stored records.
