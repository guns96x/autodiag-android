# Transport and Adapter Layer Implementation Plan

> **For agentic workers:** Execute after the foundation gate passes.

**Goal:** Implement interchangeable Bluetooth Classic SPP, BLE and USB transports plus ELM/STN/vLinker adapter engines with capability probing and deterministic fake implementations.

**Architecture:** Transport moves bytes and owns connection lifecycle. Adapter layer translates device command dialect/capabilities. Automotive protocol code is forbidden in transport modules.

**Tech Stack:** Android Bluetooth/RFCOMM, Android BLE, USB Host, Coroutines/Flow, fake transport testkit.

**Spec:** `docs/superpowers/specs/2026-09-25-autodiag-android-master-design.md`

## Review Focus
- permission denial;
- disconnect mid-read;
- BLE MTU smaller than request chunk;
- clone ELM missing claimed commands;
- adapter response without terminal prompt.

---

### Task 1: Transport API and fake transport

**Files:**
- Create: `transport-api/src/main/kotlin/.../Transport.kt`
- Create: `.../TransportTarget.kt`
- Create: `.../TransportState.kt`
- Create: `.../TransportCapabilities.kt`
- Create: `testkit/src/main/kotlin/.../FakeTransport.kt`
- Test: `transport-api/src/test/.../TransportContractTest.kt`

**Interfaces:**
```kotlin
interface Transport {
    suspend fun connect(target: TransportTarget)
    suspend fun disconnect()
    suspend fun write(data: ByteArray)
    val incoming: Flow<ByteArray>
    val state: StateFlow<TransportState>
    suspend fun capabilities(): TransportCapabilities
}
```

- [ ] Write contract tests for connect, write/incoming, disconnect, cancellation and reconnect state.
- [ ] Implement API and FakeTransport with scripted frames/faults.
- [ ] Run tests and commit `feat(transport): define transport contract and fake transport`.

### Task 2: Bluetooth Classic SPP

**Files:**
- Create module `transport-bluetooth-spp`
- Create: `BluetoothSppTransport.kt`
- Create: `BluetoothPermissionGate.kt`
- Test with Android/Robolectric fakes.

- [ ] Test permission denied, socket connect timeout, normal byte flow, remote disconnect, reconnect.
- [ ] Implement RFCOMM/SPP UUID transport with IO dispatcher and cancellation-safe close.
- [ ] Verify no adapter parsing exists in module.
- [ ] Commit `feat(transport): add Bluetooth SPP transport`.

### Task 3: BLE transport

**Files:**
- Create module `transport-ble`
- Create: `BleTransport.kt`
- Create: `BleProfile.kt`
- Create: `BleChunker.kt`

- [ ] Test service discovery, characteristic selection by profile, notification subscription, MTU chunking and disconnect.
- [ ] Implement profile-driven BLE transport.
- [ ] Commit `feat(transport): add profile-driven BLE transport`.

### Task 4: USB transport

**Files:**
- Create module `transport-usb`
- Create: `UsbTransport.kt`
- Create: `UsbDeviceMatcher.kt`
- Create: `UsbPermissionGate.kt`

- [ ] Test permission denial, endpoint discovery, device detach, read/write and cancellation.
- [ ] Implement USB Host byte transport without automotive semantics.
- [ ] Commit `feat(transport): add USB Host transport`.

### Task 5: Adapter API and scripted fake

**Files:**
- Create: `adapter-api/src/main/kotlin/.../VehicleAdapter.kt`
- Create: `.../AdapterCapabilities.kt`
- Create: `.../AdapterProtocolProfile.kt`
- Create: `testkit/.../FakeVehicleAdapter.kt`

**Interface:**
```kotlin
interface VehicleAdapter {
    suspend fun initialize()
    suspend fun probeCapabilities(): AdapterCapabilities
    suspend fun configure(profile: AdapterProtocolProfile)
    suspend fun transmit(frame: AdapterFrame): AdapterResponse
    suspend fun reset()
}
```

- [ ] Test capability downgrade when a command returns unsupported.
- [ ] Implement interface/fake.
- [ ] Commit `feat(adapter): define adapter contract and capability model`.

### Task 6: ELM/STN/vLinker engines

**Files:**
- Create modules `adapter-elm`, `adapter-stn`, `adapter-vlinker`
- Create shared parser utilities in `adapter-api`.

- [ ] Add transcript fixtures for known-good ELM, incomplete clone, STN extensions, vLinker extensions.
- [ ] Implement init/reset, prompt framing, echo/spacing normalization, firmware identity and command probing.
- [ ] Represent unsupported capability explicitly rather than failing open.
- [ ] Run all adapter tests.
- [ ] Commit separate commits for ELM, STN and vLinker.

### Gate
All transports must pass the same contract tests. All adapter families must produce explicit capability objects and never assume CAN/K-Line/raw-frame features from product name alone.
