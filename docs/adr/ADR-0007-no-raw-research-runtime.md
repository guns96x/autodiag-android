# ADR-0007: No raw research JSON at runtime

Status: Accepted.

Decision: Android runtime never consumes ghidracarista research files directly.

Consequences: all research data passes schema, provenance, conflict, applicability and protocol validation before pack compilation.