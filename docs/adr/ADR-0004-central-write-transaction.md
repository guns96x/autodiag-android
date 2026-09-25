# ADR-0004: Central write transaction

Status: Accepted.

Decision: every coding/adaptation/service write is mediated by WriteTransactionEngine.

Consequences: backup, prechecks, write, read-back, recovery and transaction persistence cannot be bypassed by UI or feature runtimes.