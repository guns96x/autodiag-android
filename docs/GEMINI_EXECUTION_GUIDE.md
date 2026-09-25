# Gemini Execution Guide

**Status:** NORMATIVE.

Gemini implements; it does not redesign.

## Start order

1. Read `docs/superpowers/specs/2026-09-25-autodiag-android-master-design.md`.
2. Read every NORMATIVE document under `docs/architecture`, `docs/runtime-pack`, `docs/evidence`, and `docs/testing`.
3. Read `docs/superpowers/plans/2026-09-25-autodiag-android-master-plan.md`.
4. Execute subsystem plans in the exact order listed there.
5. Work task-by-task, tests first.

## Hard rules

- Do not change architecture because another implementation seems easier.
- Do not invent protocol bytes, ECU addresses, DID/channel layouts, whitelist contents, security procedures or success criteria.
- Missing automotive fact -> `BLOCKED_EVIDENCE`.
- Do not use array order to resolve variants.
- Do not create executable PARTIAL/UNRESOLVED/CONFLICT variants.
- Do not bypass protected authorization.
- Do not let UI send raw protocol writes.
- Do not skip a phase gate.

## Task loop

For each task:
1. inspect referenced contracts;
2. write failing test;
3. run focused test and record expected failure;
4. implement;
5. run focused test;
6. run module tests;
7. run relevant evidence/protocol validator;
8. update generated docs/reports if affected;
9. commit one logical change;
10. push main;
11. continue only when green.

## Evidence blockage

A blocked feature does not block building the rest of the app. Implement the generic runtime and classify the feature. Add exact blocker to generated blocked-evidence report.

## Architecture conflict

If two NORMATIVE documents conflict:
1. stop that task only;
2. do not guess;
3. record exact file/section conflict in `docs/UNRESOLVED_DECISIONS.md`;
4. continue independent tasks;
5. do not silently choose one.

## Completion claim

Do not claim 100% based on UI/features count. Use generated coverage states and final acceptance artifacts defined in the master spec.
