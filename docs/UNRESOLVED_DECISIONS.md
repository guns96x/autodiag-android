# Unresolved Decisions

**Status:** NORMATIVE TRACKER.

## Software architecture

None.

Gemini is not expected to make architectural/library/API/module-lifetime decisions outside the committed normative specifications.

## BLOCKED_EVIDENCE

Current known evidence blockers are defined canonically in `docs/evidence/EVIDENCE_REGISTER.md`.

Important current blockers include:
- TP2.0 channel setup/addressing conflict and missing parameter/sequence parsing evidence;
- KWP coding/adaptation payload parser/layout gaps;
- UDS coding DID conflict;
- unresolved whitelist contents;
- non-reproducible/heuristic customization extraction from current molecular dataset;
- selected service-tool flows with incomplete requests/preconditions;
- selected live-data definitions without exact parser provenance.

These are automotive evidence gaps, not software architecture decisions.
