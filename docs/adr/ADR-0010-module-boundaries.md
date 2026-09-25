# ADR-0010: Module-boundary extensions

Status: Accepted.

Decision: add :core-common, :usecase and :tools:trace-toolchain beyond the original master-spec module list.

Rationale:
- :core-common owns dependency-free cross-cutting primitives.
- :usecase owns application orchestration above runtimes.
- :tools:trace-toolchain separates protocol/trace tooling from evidence compilation.

Consequences: exact allowed/forbidden dependencies are normative in MODULE_DEPENDENCY_GRAPH.md and module-dependencies.json.