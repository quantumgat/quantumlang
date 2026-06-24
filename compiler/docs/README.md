# qtlc1 Documentation Index

Status: planning contract before qtlc1 implementation

`qtlang/compiler` is the qtlc1 package: the first QuantumLang-authored
compiler built by qtlc0.

This documentation set is the implementation contract for qtlc1. Do not start
new compiler code before updating the relevant page here.

## Pages

```text
QTLC1_IMPLEMENTATION_PLAN.md
  complete qtlc1 scope, non-goals, stage order, and success criteria

QTLC1_FULL_IMPLEMENTATION_SCOPE.md
  hard qtlc1 scope lock: qtlc1 is compiler-only, required modules,
  forbidden runtime/stdlib/tooling work, Quantum style rules, and stage split

QTLC1_CORE_TYPE_MEMORY_OPERATOR_SCOPE.md
QTL_RANGE_LITERAL_MODEL.md
  qtlc0-verified primitive type surface, qtlc1 core memory facts, operator
  scope, literal typing, and what stays out of qtlc1 runtime/stdlib work

QTLC1_CONTROL_FLOW_AND_LOOP_SCOPE.md
  normal control flow, loop, parallel loop, quantum control-flow, qwhile/qloop,
  and qtlc2-builder loop boundaries

DIRECTORY_AND_MODULE_CONTRACT.md
  package/module layout rules and current directory ownership

IMPLEMENTATION_SLICES.md
  ordered buildable slices from source/diagnostics through backend proof

QUANTUM_FIRST_COMPILER_PLAN.md
  first-class quantum language architecture, not just backend emission

PROOF_AND_BUILD_GATES.md
  qtlc0 check/build proof, qtlc1 self-host proof, and blocked claims

INCREMENTAL_BUILD_AND_CACHE_PLAN.md
  magic-fast build architecture, cache layers, scheduler, and proof contracts

MAGIC_FAST_BUILD_GAP_ANALYSIS.md
  gap analysis against existing docs plus query database, daemon, qcap/qidx,
  symbol-level, and benchmark proof plan for faster-than-normal edit builds

QTLC1_IMPL_COMMAND_ROADMAP.md
  short command list for completing qtlc1 from stable IDs through qtlc2 proof

QTLC1_INDEPENDENT_NATIVE_SELF_HOSTING_ROADMAP.md
  authoritative architecture prerequisite plus eight-phase execution plan for
  qtlc1-owned HIR, MIR, machine IR, native lowering, ELF objects, linking,
  qtlc2/qtlc3 proof, 29 top-level ownership boundaries, dependency rules,
  migration commands, and the 142-page native implementation inventory

QTLC1_OVERLOAD_GENERIC_RESOLUTION_DESIGN.md
  audited overload, constructor, generic inference, extension, operator,
  ranking, diagnostic, cache, and HIR handoff contracts for real type checking

QTLC1_CANDIDATE_INDEX_DESIGN_AUDIT.md
  centralized free/member/constructor/role/extension/operator candidate
  indexes, visibility, ownership, invalidation, and complexity contract

QTLC1_NORMAL_PROJECT_COMPILATION_IMPL_COMMANDS.md
  complete global-cache, query database, per-body compilation, qtlcd, proof,
  and qtlc1-builds-qtlc2 command sequence before toolchain assembly

QTLC1_CORE_DATA_IMPL_COMMANDS.md
  command list for replacing scalar-only compiler state with compiler-owned
  Arena/Vec/Map/Index/NameId contracts without depending on runtime/stdlib

QUANTUM_OOP_SURFACE_BOUNDARY_DESIGN.md
  final OOP surface model: Type/class/resource/actor/service, impl/view,
  role/extend, bounded blanket extensions, context/extract, unsafe/extern,
  boundary rules, and BSON/complex examples

QTLC1_QUANTUM_STYLE_SOURCE_AUDIT.md
  per-directory audit for rewriting qtlc1 source itself in Quantum style:
  constructors, impl view, role/extend, context, and extract
```

## Current Package Root

```text
qtlang/compiler/
  quantum.toml
  mod.qn
  main.qn
```

Build/check target:

```bash
qtlc/build/compiler/driver/qtlc check qtlang/compiler
qtlc/build/compiler/driver/qtlc build -j4 qtlang/compiler
```

Current proof status:

```text
per-module qtlc0 check: passed
full qtlc0 check: passed, 251 files
qtlc0 scaffold build: passed
POST-QTLC1-MAGIC-FAST-001: passed
POST-QTLC1-MAGIC-FAST-002: passed
POST-QTLC1-MAGIC-FAST-003: passed
POST-QTLC1-MAGIC-FAST-004: passed
POST-QTLC1-MAGIC-FAST-005: passed
POST-QTLC1-MAGIC-FAST-006: passed
POST-QTLC1-MAGIC-FAST-007: passed
POST-QTLC1-MAGIC-FAST-008: passed
POST-QTLC1-MAGIC-FAST-009: passed
POST-QTLC1-MAGIC-FAST-010: passed
POST-QTLC1-MAGIC-FAST-011: passed
POST-QTLC1-INCREMENTAL-QUERY-ENGINE-001: passed
POST-QTLC1-SOURCE-STYLE-002: passed
POST-QTLC1-SCOPE-STYLE-LOCK-001: passed
POST-QTLC1-CORE-TYPE-MEMORY-OPERATOR-SCOPE-001: passed
POST-QTLC1-CONTROL-FLOW-LOOP-SCOPE-001: passed
POST-QTLC0-APP-BINARY-ARTIFACT-CONTRACT-001: passed
qtlc1 frontend/parser/compiler behavior: not started
qtlc1 self-host proof: not started
```
