# qtlc1 Normal Project Compilation Implementation Commands

Status: active implementation contract

Current implementation:

```text
POST-QTLC1-GLOBAL-CACHE-CONTRACT-001: implemented
POST-QTLC1-CACHE-OBJECT-FORMATS-002: implemented
POST-QTLC1-CACHE-ACTION-CONTENT-HASH-003: implemented
POST-QTLC1-CACHE-ATOMIC-SAFETY-004: implemented
POST-QTLC1-BUILDSTAMP-GLOBAL-005: implemented
POST-QTLC1-STABLE-SEMANTIC-IDS-006: implemented
POST-QTLC1-QUERY-KEY-MODEL-007: implemented
POST-QTLC1-QDB-BINARY-STORAGE-008: implemented
POST-QTLC1-QDB-REVISION-JOURNAL-009: implemented
POST-QTLC1-PRODUCT-STORE-010: implemented
POST-QTLC1-DEPENDENCY-GRAPH-011: next
```

Purpose: qtlc1 builds qtlc2 as a normal project before qtlc2/qtlc3 assemble
the complete toolchain.

```text
qtlc0 -> qtlc1
qtlc1 -> qtlc2 using ~/.cache/qtlc/
qtlc2/qtlc3 -> toolchain using ~/.quantumlang/qstore/
```

## Locked Artifact Ownership

```text
normal project cache:
  ~/.cache/qtlc/

toolchain/self-host component store after qtlc2 proof:
  ~/.quantumlang/qstore/

installed trusted toolchain:
  ~/.quantumlang/toolchains/<channel>/

project build directory:
  requested final delivery artifacts only
```

The normal cache is a global file-based content-addressed store:

```text
~/.cache/qtlc/
  00..ff/<content-hash>.<qcm|qco|qca|qcx|qcd>
  actions/00..ff/<action-hash>.qci
  cache.version
  cleanup.lock
  .tmp/
  .leases/
  .quarantine/
```

Project names and source paths are diagnostic metadata only. They are never
cache directory names or reusable product identities.

Public artifact rules:

```text
.qcap  compiler API/type/generic/ABI capsule
.qidx  mmap module/symbol index
.qna   Quantum multi-target native archive container
.qpkg  final package distribution archive
.qnpkg bootstrap compatibility metadata only
```

`.qn.buildstamp` is not a public artifact. Build-result state is a `.qcm`
product reached through a `.qci` action.

## Foundation

```text
POST-QTLC1-GLOBAL-CACHE-CONTRACT-001
  lock global normal-cache ownership, CAS buckets, clean project outputs,
  hashed workspace identity, and qtlc1-to-qtlc2 normal-build policy

POST-QTLC1-CACHE-OBJECT-FORMATS-002
  define .qci, .qcm, .qco, .qca, .qcx, and .qcd binary contracts

POST-QTLC1-CACHE-ACTION-CONTENT-HASH-003
  separate action and content hashes; use domain-separated 256-bit keys

POST-QTLC1-CACHE-ATOMIC-SAFETY-004
  atomic writes, active leases, quarantine, validation, schema migration

POST-QTLC1-BUILDSTAMP-GLOBAL-005
  replace project .qn.buildstamp with globally indexed build-result metadata

POST-QTLC1-STABLE-SEMANTIC-IDS-006
  stable package/module/file/symbol/body/node/generic/object/link IDs

POST-QTLC1-QUERY-KEY-MODEL-007
  complete query keys, component epochs, target-independent frontend keys

POST-QTLC1-QDB-BINARY-STORAGE-008
  mmap compact query/product/dependency graph snapshots without external DB

POST-QTLC1-QDB-REVISION-JOURNAL-009
  crash-safe revisions and continuous completed-product commits

  implemented contract:
  - immutable revision rows with parent revision and snapshot identity
  - append-only completed-product commit rows
  - hash-chained durable commits
  - atomic publication of only sealed revisions
  - recovery plans for published state, durable replay, incomplete-tail
    truncation, and corrupt-journal rejection
  - QDB snapshot and journal remain compiler-owned global `.qcm` metadata
    products; optional remote storage remains a later plugin boundary

POST-QTLC1-PRODUCT-STORE-010
  real global CAS lookup, validation, write-back, and hot-memory products

  implemented contract:
  - typed global CAS addresses under `~/.cache/qtlc/<bucket>/<hash>.<format>`
  - action-to-content links remain domain separated
  - object header, schema, length, content identity, and checksum validation
  - validated hot-memory products take precedence over global file products
  - atomic write-back requires active lease, synced temporary payload,
    atomic rename, schema compatibility, and validation
  - successful durable writes update ProductStore and append a completed
    QDB journal commit in one BuildDatabase transition
  - invalid products are rejected without entering the content or hot store

POST-QTLC1-DEPENDENCY-GRAPH-011
  forward/reverse query edges, cycle detection, exact invalidation reasons

POST-QTLC1-DIRTY-QUERY-SET-012
  typed dirty QueryKey sets for source/symbol/body/target/manifest changes

POST-QTLC1-RED-GREEN-EXECUTOR-013
  replay green products and execute only red queries

POST-QTLC1-QUERY-SCHEDULER-014
  dependency-ready work stealing, critical path, cancellation, memory budget
```

## Frontend And Types

```text
POST-QTLC1-SOURCE-SNAPSHOT-QUERY-015
  source snapshots with metadata hints and verified content fingerprints

POST-QTLC1-TOKEN-QUERY-016
  lex changed files only; cache tokens, trivia, and diagnostics

POST-QTLC1-GREEN-SYNTAX-QUERY-017
  persistent green syntax and changed-region reparsing

POST-QTLC1-EXPORT-SURFACE-QUERY-018
  stable import/export and public API fingerprints

POST-QTLC1-SYMBOL-RESOLUTION-QUERY-019
  symbol-use resolution and exact imported-symbol edges

POST-QTLC1-TYPE-SIGNATURE-QUERY-020
  separate declaration/callable signatures from implementation bodies

POST-QTLC1-TYPED-BODY-QUERY-021
  one typed product and diagnostic set per body

POST-QTLC1-LAYOUT-ABI-QUERY-022
  cached type layout and ABI products with exact dependents

POST-QTLC1-GENERIC-INSTANCE-QUERY-023
  typed/HIR/MIR/native products keyed by generic arguments and policy

POST-QTLC1-DIAGNOSTIC-PRODUCT-024
  replayable diagnostics per query with source-span dependencies
```

## IR And Native Backend

```text
POST-QTLC1-HIR-BODY-QUERY-025
  one HIR product per BodyId

POST-QTLC1-MIR-BODY-QUERY-026
  one MIR/CFG product per BodyId

POST-QTLC1-OPTIMIZED-MIR-QUERY-027
  profile/policy-specific optimized MIR product

POST-QTLC1-CODEGEN-UNIT-POLICY-028
  per-body object units with stable grouping for tiny bodies

POST-QTLC1-NATIVE-OBJECT-QUERY-029
  emit changed .qco units and restore unchanged global objects

POST-QTLC1-PACKAGE-ARCHIVE-QUERY-030
  hidden .qca from ordered object hashes

POST-QTLC1-INCREMENTAL-LINK-QUERY-031
  link action over ordered object/archive hashes; restore unchanged .qcx
```

## Interfaces And Packages

```text
POST-QTLC1-QCAP-QUERY-032
  compiler-readable API/type/generic/ABI capsule

POST-QTLC1-QIDX-QUERY-033
  mmap module/symbol index

POST-QTLC1-QNA-CONTAINER-034
  multi-target native archive container, not a duplicate platform .a

POST-QTLC1-QPKG-DISTRIBUTION-035
  final .qpkg distribution; bootstrap .qnpkg read compatibility
```

## Hot Compiler

```text
POST-QTLC1-QTLCD-WORKSPACE-036
  hot source table, graph, query products, capsules, objects, diagnostics

POST-QTLC1-QTLCD-FILE-WATCH-037
  exact dirty-file notifications without full workspace scanning

POST-QTLC1-QTLCD-HOT-BUILD-038
  request linkedTarget(qtlc2) and execute only red queries
```

## Proof Gates

```text
POST-QTLC1-NO-CHANGE-PROOF-039
  zero lex/parse/check/lower/emit/link

POST-QTLC1-FORMATTING-CHANGE-PROOF-040
  source/line-map update only

POST-QTLC1-ONE-BODY-CHANGE-PROOF-041
  one body check, HIR, MIR, object, and relink

POST-QTLC1-PUBLIC-API-CHANGE-PROOF-042
  exact exported-symbol dependents only

POST-QTLC1-CROSS-PROJECT-CAS-PROOF-043
  reuse identical products across independent project roots

POST-QTLC1-INTERRUPTED-BUILD-RESUME-044
  retain every atomically completed product

POST-QTLC1-PARALLEL-BUILD-PROOF-045
  worker utilization, critical path, no global compiler locks

POST-QTLC1-BUILDS-QTLC2-046
  real qtlc1 frontend/type/HIR/MIR/native/link produces qtlc2

POST-QTLC1-QTLC2-DETERMINISM-047
  repeated source produces identical deterministic product hashes
```

## Toolchain Work After qtlc2

```text
POST-QTLC2-QSTORE-001
  implement ~/.quantumlang/qstore/

POST-QTLC2-TOOLCHAIN-BUILD-002
  build compiler, core, runtime, stdlib, platforms, and plugins

POST-QTLC2-TOOLCHAIN-INSTALL-003
  install trusted artifacts under ~/.quantumlang/toolchains/<channel>/

POST-QTLC3-TOOLCHAIN-PACKAGE-004
  signed capsules, indexes, native libraries, plugins, and target sysroots
```
