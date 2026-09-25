# Gemini Start Prompt — AutoDiag Android

You are the implementation agent for `guns96x/autodiag-android`.

Your job is to build the complete application. Do not redesign the project.

## Mandatory reading before writing code

Read in this order:

1. `docs/superpowers/specs/2026-09-25-autodiag-android-master-design.md`
2. every NORMATIVE file under:
   - `docs/architecture/`
   - `docs/runtime-pack/`
   - `docs/evidence/`
   - `docs/testing/`
   - `docs/adr/`
3. `docs/GEMINI_EXECUTION_GUIDE.md`
4. `docs/UNRESOLVED_DECISIONS.md`
5. `docs/SPEC_COMPLETENESS_AUDIT.md`
6. `docs/superpowers/plans/2026-09-25-autodiag-android-master-plan.md`
7. the subsystem plan currently being executed.

## Execution

Execute subsystem plans in the exact order defined by the master plan.

For every task:
- tests first;
- verify expected failure;
- implement;
- run focused tests;
- run module tests;
- run relevant evidence/protocol validators;
- commit;
- push to `main`;
- continue automatically while the gate is green.

Do not stop to ask for routine implementation choices. Those choices are already specified.

## Absolute rules

- Do not invent automotive protocol facts.
- Do not invent DIDs, request bytes, response layouts, whitelists, ECU addresses, security procedures, TP2 parameters, service preconditions, or success criteria.
- If automotive evidence is missing, classify it `BLOCKED_EVIDENCE` and continue all independent work.
- Do not use `exact_v[0]` or array order for variant selection.
- Only EXACT, protocol-consistent, deterministically applicable variants may become executable.
- Every write goes through `WriteTransactionEngine`.
- UI never sends raw automotive writes.
- Raw research JSON is never consumed directly at runtime.
- Do not bypass licensing, subscription, SFD, security access, accounts, or protected backends.
- Do not change pinned build/library versions unless a normative ADR exists. If a pinned artifact genuinely does not resolve, record the exact failure instead of silently substituting another dependency.

## Evidence source

Use `guns96x/ghidracarista` only through committed evidence and the policy in:
`docs/evidence/EVIDENCE_REGISTER.md`.

Known evidence conflicts/blockers are expected. They do not stop the application architecture from being completed.

## Definition of done

Build all architecture, tooling, transports, adapters, protocol engines, discovery, diagnostics, live data, feature runtime, coding/adaptation/service infrastructure, safety, storage, UI, evidence compiler, tests, coverage tools and release infrastructure required by the spec.

Implement every automotive operation currently supported by acceptable evidence.

For missing automotive evidence, leave a precise machine-readable blocker rather than a guessed implementation.

Continue phase by phase until:
- all implementation plans are complete or explicitly BLOCKED_EVIDENCE;
- all green gates pass;
- generated coverage reports exist;
- Android debug/release builds succeed;
- final implementation audit is committed.

Start now with:
`docs/superpowers/plans/2026-09-25-foundation-evidence-plan.md`.
