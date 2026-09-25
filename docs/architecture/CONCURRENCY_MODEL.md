# Concurrency Model

**Status:** NORMATIVE.

## Scopes

- App scope: process lifetime, SupervisorJob + default dispatcher.
- VehicleSession scope: created on successful adapter connection; cancelled on explicit disconnect/session teardown.
- ViewModel scopes: UI only; never own protocol sessions.
- Operation scope: child of VehicleSession for diagnostics/live data.
- Write/service scope: child of VehicleSession and guarded by exclusive mutex.

## Dispatchers

- blocking Bluetooth/USB/GATT callbacks -> IO bridge;
- protocol parsing/state machines -> Default;
- Room -> Room-managed suspend APIs;
- Compose/ViewModel state publication -> Main.immediate.

No `GlobalScope`.

## Exclusive operations

One `Mutex` per VehicleSession:
- coding write;
- adaptation write;
- basic setting;
- service routine.

LiveDataScheduler transitions to PAUSED_EXCLUSIVE_OPERATION before mutex grant and resumes after cleanup.

## Cancellation

Cancellation propagates downward. Cleanup steps execute in `withContext(NonCancellable)`. Cancellation never converts an uncertain post-write state into success.

## Reconnect

Reconnect creates a new protocol channel and re-identifies the ECU. Existing protocol channel objects are never reused after physical disconnect.
