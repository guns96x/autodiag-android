# ADR-0001: Data-driven runtime packs

Status: Accepted.

Decision: protocol/runtime code is generic; feature/vehicle behavior enters the app through validated, immutable, signed runtime data packs.

Consequences: research JSON is compiled, not loaded directly; pack schema/versioning/validation are release-critical.