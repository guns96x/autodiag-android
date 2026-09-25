# Vehicle Detection, Diagnostics and Live Data Implementation Plan

> **For agentic workers:** Execute after protocol gate.

**Goal:** Build the read-only vertical product path from adapter connection through vehicle/ECU identification, AutoScan, DTCs, ECU info and live data.

**Architecture:** OEM VAG supplies discovery/identity definitions; generic runtimes execute operation plans and produce canonical identities/results.

**Spec:** `docs/superpowers/specs/2026-09-25-autodiag-android-master-design.md`

## Review Focus
- partial gateway response;
- ECU present but identity query fails;
- duplicate ECU logical address across profiles;
- live-data value outside valid range;
- disconnect during AutoScan.

---

### Task 1: VAG ECU/address profile registry

**Files:**
- `oem-vag/.../VagEcuAddress.kt`
- `oem-vag/.../VagProtocolProfile.kt`
- `oem-vag/.../VagDiscoveryDefinitions.kt`

- [ ] Add tests from verified evidence for known ECU addressing/discovery examples.
- [ ] Implement explicit registry without generic fallback.
- [ ] Unknown ECU returns unresolved, never a default address.
- [ ] Commit `feat(vag): add evidence-backed VAG addressing profiles`.

### Task 2: Vehicle/ECU discovery

**Files:**
- `vehicle-detection/.../VehicleDetector.kt`
- `ecu-discovery/.../EcuDiscoveryEngine.kt`
- `oem-vag/.../VagGatewayDiscovery.kt`

- [ ] Add trace fixtures for UDS gateway list and TP2 gateway list.
- [ ] Implement protocol probe -> gateway -> ECU list pipeline.
- [ ] Preserve raw discovery responses in trace output.
- [ ] Commit `feat(discovery): add VAG vehicle and ECU discovery`.

### Task 3: ECU identity engine

**Files:**
- `vehicle-detection/.../EcuIdentityReader.kt`
- `oem-vag/.../VagIdentityDefinitions.kt`

- [ ] Add fixtures for VIN, part number, SW/HW, serial, component, ASAM/ODX, slave IDs.
- [ ] Implement query plans by protocol/profile.
- [ ] Do not infer missing identity fields.
- [ ] Commit `feat(identity): add ECU identity pipeline`.

### Task 4: Diagnostics runtime

**Files:**
- `diagnostics-runtime/.../AutoScanEngine.kt`
- `.../DtcReader.kt`
- `.../DtcClearer.kt`
- `.../ScanResult.kt`

- [ ] Add UDS and TP2 DTC golden fixtures.
- [ ] Implement full scan with per-ECU error isolation.
- [ ] Implement clear only through explicit operation plans.
- [ ] Add before/after scan comparison.
- [ ] Commit `feat(diagnostics): add AutoScan and DTC runtime`.

### Task 5: Live-data model and scheduler

**Files:**
- `live-data-runtime/.../LiveDataParameter.kt`
- `.../LiveDataScheduler.kt`
- `.../LiveDataDecoder.kt`

- [ ] Add tests for signedness, endian, scaling, range rejection, grouping and backpressure.
- [ ] Implement definition-driven decoding and freshness timestamps.
- [ ] Suspend/degrade polling during exclusive operations.
- [ ] Commit `feat(livedata): add parameter decoder and scheduler`.

### Task 6: Read-only integration fixture

**Files:**
- `testkit/src/testFixtures/.../Pq35Golf5BlsFixture.kt`
- `app/src/androidTest/.../ReadOnlyVerticalFlowTest.kt`

- [ ] Script adapter -> discovery -> identity -> scan -> DTC -> live data.
- [ ] Run entirely through FakeTransport/trace replay.
- [ ] Assert no write operation is reachable.
- [ ] Commit `test: add PQ35 read-only vertical replay`.

### Gate
The complete read-only flow works in replay before any coding/adaptation/service write is enabled.
