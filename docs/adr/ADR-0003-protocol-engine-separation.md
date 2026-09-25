# ADR-0003: Separate protocol engines

Status: Accepted.

Decision: CAN, ISO-TP, UDS, KWP2000 and TP2.0 are separate modules/state machines.

Consequences: OEM feature definitions configure engines but do not reimplement protocol state machines.