# ADR-0008: Ambiguity blocks write

Status: Accepted.

Decision: if multiple equally applicable exact variants remain, selection result is AMBIGUOUS and write is blocked.

Consequences: source order, variant ID and array order are never fallback priorities.