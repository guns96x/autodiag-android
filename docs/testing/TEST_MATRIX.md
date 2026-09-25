# Test Matrix

**Status:** NORMATIVE.

| Area | Unit | Golden | Replay | Fault injection | Android integration | Real vehicle |
|---|---|---|---|---|---|---|
| Domain/evidence | required | n/a | n/a | malformed pack | loader activation | n/a |
| Transport | required | transcript | required | disconnect/timeout | device/emulator API behavior | adapter smoke |
| Adapter | required | AT/ST transcript | required | unsupported/broken command | connection flow | adapter capability probe |
| ISO-TP | required | required | required | seq/FC/timeout | n/a | where host-side used |
| UDS | required | required | required | NRC/pending/timeout | n/a | read-only baseline |
| KWP/TP2 | required | required before executable | required | ACK/sequence/timeout | n/a | PQ35 required |
| Discovery/identity | required | request/response fixtures | required | partial ECU failure | app flow | PQ35 baseline |
| DTC | required | UDS/KWP | required | malformed/timeout | screen flow | PQ35 baseline |
| Live data | required | parser vectors | required | out-of-range/backpressure | screen flow | evidenced parameters |
| Variant resolution | required | n/a | vehicle fixture | ambiguity/conflict | feature screen | validation target |
| Coding/adaptation | required | required | required | mismatch/disconnect | confirmation flow | only exact low-risk target |
| Service | required | routine vectors | required | pending/failure/cancel | wizard/progress | evidenced procedure |
| Storage | required | n/a | n/a | migration corruption | Room migration | n/a |
| UI | state tests | n/a | fake data | errors/unavailable | Compose tests | n/a |

## Canonical commands

- JVM module: `./gradlew :<module>:test`
- Android unit: `./gradlew :<module>:testDebugUnitTest`
- app instrumentation: `./gradlew :app:connectedDebugAndroidTest`
- full local gate: command from `BUILD_BASELINE.md` §9.
- evidence validation: `./gradlew :tools:evidence-toolchain:run --args="validate-all --pack <path>"`
- trace replay: `./gradlew :tools:trace-toolchain:run --args="replay --fixture <path>"`

Every implementation task adds a focused test command to its commit description if the subsystem plan did not already name one.
