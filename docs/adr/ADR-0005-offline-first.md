# ADR-0005: Offline-first

Status: Accepted.

Decision: all functionality that does not technically require external authorization works locally.

Consequences: backend dependency from the reference application is not reproduced unless a legitimate external authorization is inherently required.