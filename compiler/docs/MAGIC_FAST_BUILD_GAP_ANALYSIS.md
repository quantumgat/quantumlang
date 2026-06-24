# qtlc1 Magic-Fast Build Gap Analysis

Status: required architecture closure before qtlc1 frontend implementation

Goal: qtlc1 builds should feel instant when nothing important changed, and
should rebuild only the smallest provable unit after an edit. This is a design
target, not a marketing claim. It must be proven by build reports and repeatable
benchmarks.

## Existing Docs Already Say

The older design docs already contain the right ingredients, but they are spread
across several places:

```text
toolchain-3.md
  mmap .qcap/.qidx
  qstore content cache
  symbol-level dependencies
  compile changed functions only
  generic instantiation cache
  qtlcd compiler daemon

docs/compiler_algorithms/INCREMENTAL_CACHE_QUERY_GRAPH.md
  query keys
  dependency edges
  red/green invalidation
  product cache validation
  incremental-json proof dump

docs/QTLC_CLEAN_ARCHITECTURE_BLUEPRINT.md
  CompilationSession
  PipelineProducts
  explicit product store
  pass inputs/outputs

docs/complete/self_hosting/compiler_driver_product_architecture.md
  product directories
  query-friendly pipeline
  incremental/cache layout

docs/tooling/WATCH_MODE_COMPILER_SERVER.md
  compiler server
  watch mode
  changed-product reporting
```

So the concept is not missing. The gap is that qtlc1 must make these ideas the
main compiler architecture from the first source slice.

## Main Gap

The current qtlc1 cache plan lists cache layers, but it does not yet make the
query database the center of the compiler.

Wrong shape:

```text
run lex
run parse
run resolve
run type check
run lower
run emit
then try to cache outputs
```

Correct qtlc1 shape:

```text
requested output
  -> query database asks only for required facts
  -> dependency graph validates fingerprints
  -> product store replays green products
  -> executor recomputes only red queries
  -> build report proves every reuse decision
```

qtlc1 should be a demand-driven compiler. Passes still exist, but they are query
providers, not a mandatory whole-package pipeline.

## Required Architecture

qtlc1 needs these core pieces before serious frontend work grows:

```text
BuildDatabase
  owns query execution, revisions, dependency edges, and product lookup

QueryKey
  stable identity for one compiler fact

QueryResult
  deterministic product or diagnostic failure

DependencyEdge
  records which query result depended on which input fingerprint

ProductStore
  durable on-disk products plus in-memory hot products

DirtySet
  changed source/options/target/package facts for current revision

CacheReport
  user-visible proof of hits, misses, rejects, and rebuild reasons
```

Minimum query examples:

```text
sourceFingerprint(fileId)
tokens(fileId)
ast(fileId)
exportSurface(moduleId)
importGraph(packageId)
resolveSymbol(symbolUseId)
typeSignature(itemId)
typeBody(bodyId)
hirBody(bodyId)
mirBody(bodyId)
optimizedBody(bodyId, profile)
nativeObject(codegenUnitId, target, profile)
quantumArtifact(quantumUnitId, target, provider)
linkedTarget(targetId)
```

## Stable Identity

Fast invalidation is impossible without stable IDs. qtlc1 must assign stable
IDs from semantic location and content, not from arena allocation order.

Required IDs:

```text
PackageId
ModuleId
FileId
ItemId
BodyId
SymbolId
TypeId
EffectId
QuantumResourceId
GenericInstanceId
CodegenUnitId
ObjectUnitId
LinkTargetId
```

Rule:

```text
formatting or comments:
  may change source hash and line map
  must not change item/body identity

private body edit:
  changes BodyId product fingerprint
  must not change public ItemId signature if API is same

public signature edit:
  changes export surface fingerprint
  invalidates only dependents that use that symbol
```

## Green Tree Requirement

qtlc1 parser should produce reusable green syntax nodes:

```text
green node:
  immutable syntax shape
  content hash
  child hashes
  no parent pointer

red node:
  view with parent/range/context
  cheap to recreate
```

This lets a one-line edit reuse most syntax tree nodes instead of reparsing and
reallocating the whole file as one product.

## Symbol-Level Dependencies

File-level dependency is too coarse. qtlc1 must track:

```text
bad:
  main.qn depends on std::io

good:
  main.body#12 depends on std::io::println signature
  main.body#12 does not depend on std::io::readLine
```

Dependency edge granularity:

```text
module graph edge
export symbol edge
type signature edge
body symbol-use edge
generic instance edge
ABI layout edge
object relocation edge
link input edge
```

## Source Avoidance For Dependencies

Normal app builds should not rescan dependency source. Installed packages,
stdlib, runtime, platform, and plugins are loaded from compiler-readable
capsules:

```text
sysroot.qidx
  import path -> qcap + module/symbol offset

*.qcap
  public API
  type facts
  generic IR
  ABI facts
  inlineable facts
  dependency fingerprints
```

Rule:

```text
if dependency qcap/qidx is valid:
  do not read dependency .qn source during normal app build
```

This is one major way QuantumLang can beat C++ header/include behavior and avoid
Rust-style large crate invalidation.

## Function And Body Object Units

qtlc1 native backend must avoid one huge object for a whole package.

Preferred unit:

```text
one body/function -> one object-unit key
small functions may be grouped by stable partition policy
```

Object unit key:

```text
body MIR fingerprint
type layout fingerprint
ABI fingerprint
target fingerprint
profile/optimization fingerprint
backend version
debug-info policy
```

If only one body changes, qtlc1 should re-emit only that object unit where the
ABI and partition policy allow it.

## Generic Instantiation Cache

Generic code must not be re-solved and re-emitted repeatedly.

Key:

```text
generic item id
type argument ids
effect/capability argument ids
target/profile
monomorphization policy version
```

Products:

```text
typed generic instance
HIR instance
MIR instance
native object unit
quantum lowering unit when relevant
```

## True Parallel Scheduler

`-j` must mean real work is happening on worker threads.

Scheduler rules:

```text
work stealing
dependency-ready task queue
critical-path priority
memory budget per worker
cancel/retry support
no global lock around type checking
no global lock around backend emit
```

Parallel units:

```text
lex by changed file
parse by changed file or changed syntax region
export scan by module
type signatures by independent item
typed bodies by independent body
HIR/MIR by body
generic instances by instance key
native emit by object unit
quantum transpile by circuit/unit
```

## qtlcd Hot Build Mode

Cold `qtlc build` must be correct and cache-backed. Hot builds should use
`qtlcd` for the fastest loop.

`qtlcd` keeps:

```text
workspace graph
source file table
mmap qcap/qidx handles
green syntax cache
symbol index
query database
type cache
diagnostic cache
object/link cache index
```

Expected flow:

```text
qtlcd start
edit file
qtlc build --hot
  send dirty set
  request linkedTarget(app)
  receive changed-products report
```

Daemon safety:

```text
manifest change invalidates package graph
target/profile change invalidates backend layers
compiler version change invalidates matching query layers
memory pressure evicts cold products, not dependency graph identity
```

## Why This Can Beat C++, Rust, And Go

This design can be faster only if qtlc1 actually avoids work those systems often
must do:

```text
C++ gap:
  header/include parsing causes broad source work
  qtlc uses qcap/qidx binary metadata and source avoidance

Rust gap:
  crate-level boundaries can still be large
  qtlc tracks symbol/body/generic-instance dependencies

Go gap:
  package builds are fast but package-level invalidation is still coarse
  qtlc can recheck/re-emit a single changed body when public API is unchanged

All three:
  normal command-line cold starts reload compiler state
  qtlcd keeps graph, capsules, syntax, and type facts hot
```

This does not guarantee every full cold build beats mature compilers. The
real target is edit-build-test speed: no-change, one-body-change, and
one-symbol-change builds must prove much smaller work.

## Proof Budgets

Initial qtlc1 budgets for a medium package:

```text
no-change hot build:
  target: under 200 ms
  proof: link_skipped=true and all query layers green

no-change cold build:
  target: under 1 s after cache is warm
  proof: mmap metadata and replayed products, no source dependency scan

one body edit hot build:
  target: under 500 ms for frontend plus affected backend unit
  proof: changed_bodies=1 or explained affected_count

one public API edit:
  target: proportional to actual symbol dependents
  proof: affected_dependents list, unaffected_modules_reused=true

full cold build:
  target: parallel from first real phase
  proof: workers_used matches ready work and CPU utilization is visible
```

Budgets must be measured and adjusted by hardware class. A budget miss is not
automatically a compiler bug, but an unexplained broad rebuild is a bug.

## Required Build Report

Every build should be able to print:

```text
qtlc build app --explain-cache
```

Report fields:

```text
revision
dirty_files
dirty_symbols
queries_total
queries_green
queries_red
cache_hits_by_layer
cache_misses_by_layer
reject_reasons
source_dependency_files_read
qcap_files_mmapped
typed_bodies_rechecked
object_units_reemitted
link_skipped
workers_requested
workers_used_peak
workers_used_average
critical_path_ms
wall_time_ms
```

## qtlc1 Implementation Commands

Implement in this order:

```text
POST-QTLC1-MAGIC-FAST-001
  define stable ID model and query keys

POST-QTLC1-MAGIC-FAST-002
  implement BuildDatabase, ProductStore, DirtySet, CacheReport skeleton

POST-QTLC1-MAGIC-FAST-003
  source fingerprint, token, and green-tree AST queries

POST-QTLC1-MAGIC-FAST-004
  export surface and symbol-level dependency edges

POST-QTLC1-MAGIC-FAST-005
  type signature and typed body queries

POST-QTLC1-MAGIC-FAST-006
  HIR/MIR body query products

POST-QTLC1-MAGIC-FAST-007
  generic instantiation cache

POST-QTLC1-MAGIC-FAST-008
  native object-unit cache and link cache

POST-QTLC1-MAGIC-FAST-009
  qcap/qidx mmap dependency source-avoidance proof

POST-QTLC1-MAGIC-FAST-010
  qtlcd hot-build protocol and daemon proof

POST-QTLC1-MAGIC-FAST-011
  no-change, one-body-change, public-API-change benchmark gates
```

Do not call qtlc1 "super fast" until these gates print proof.

