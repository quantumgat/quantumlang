# qtlc1 Incremental Build And Cache Plan

Status: required architecture contract before compiler implementation

Goal: qtlc1 must feel instant on normal edits. Fast build is not a feature to
patch later; every compiler layer must produce stable cacheable artifacts from
day one.

Related architecture closure:

```text
MAGIC_FAST_BUILD_GAP_ANALYSIS.md
  explains the missing query-database, stable-ID, daemon, qcap/qidx, and proof
  pieces required for builds that feel faster than C++/Rust/Go edit loops
```

## Product Targets

```text
no-change check:
  should skip lex/parse/resolve/type/lower/backend
  target user experience: near-instant

one-file body edit:
  reprocess changed file
  recheck only affected body/function
  reuse public API and dependents if signature unchanged
  re-emit only changed object/function unit

one-file public API edit:
  reprocess changed file
  recheck dependent modules whose imported API changed
  relower/re-emit only affected bodies

full cold build:
  parallel from the first real phase
  write all durable caches continuously
  never wait until the end to save progress
```

## Core Rule

Every compiler product must have:

```text
stable key
stable serialized form
input dependency list
toolchain version
target/profile key
diagnostic replay metadata
validation proof
```

If a compiler layer cannot be cached, it must explain why.

## Central Architecture Rule

qtlc1 is a demand-driven query compiler.

The compiler must not be designed as a mandatory whole-package pipeline with
cache added afterward. Each phase provides query functions, and the build asks
for the final requested product:

```text
linked target
  -> object units
  -> lowered bodies
  -> typed bodies
  -> type signatures
  -> resolved symbols
  -> AST/tokens/source fingerprints
```

The query database decides what is green, what is red, and what must be
recomputed.

Required shared services:

```text
BuildDatabase
ProductStore
DependencyGraph
DirtySet
CacheReport
WorkScheduler
```

No frontend, semantic, IR, backend, or link layer may hide dependencies outside
this database.

## Build Pipeline

```text
source
  -> tokens
  -> AST
  -> module import/export signatures
  -> name resolution
  -> type signatures
  -> typed bodies
  -> semantic module
  -> HIR
  -> MIR
  -> optimized MIR
  -> native/quantum lowered units
  -> object/artifact units
  -> package/link output
```

Each arrow is an incremental boundary.

## Cache Layers

### 1. Source Fingerprint Cache

Key:

```text
absolute package id
relative file path
file size
content hash
mtime only as hint, never as truth
edition
lexer mode
```

Output:

```text
SourceText fingerprint
line map
newline style
encoding
```

### 2. Token Cache

Key:

```text
source fingerprint
lexer version
edition
feature flags
```

Output:

```text
token stream
trivia stream
literal facts
lexer diagnostics
```

Rule:

```text
changed file only re-lexes that file
unchanged files replay tokens directly
```

### 3. AST Cache

Key:

```text
token cache key
parser version
grammar feature flags
```

Output:

```text
AST arena/tree
node ids
source ranges
parser diagnostics
top-level item index
```

Rule:

```text
body-only edits must not invalidate unrelated files
```

### 4. Import/Export Signature Cache

Key:

```text
AST top-level item signature hash
mod.qn export hash
module.toml hash when present
quantum.toml package fields
```

Output:

```text
imports
exports
public symbol names
visibility facts
module graph edges
```

Rule:

```text
if this cache is unchanged, dependent modules do not need graph rebuild
```

### 5. Name Resolution Cache

Key:

```text
module graph key
file AST key
visible export surface keys
```

Output:

```text
resolved symbol ids
scope facts
overload candidate sets
unresolved-name diagnostics
```

### 6. Type Signature Cache

Key:

```text
resolved declaration key
generic parameter key
type checker version
builtin type catalog key
quantum type catalog key
```

Output:

```text
function signatures
struct/data shape fields
constructor signatures
method signatures
effect/capability signatures
public API type facts
```

Rule:

```text
if type signature unchanged, downstream dependents avoid recheck
```

### 7. Typed Body Cache

Key:

```text
body AST key
local signature key
visible type signature keys
effect/capability policy key
```

Output:

```text
typed expression nodes
typed statement nodes
local binding types
body diagnostics
effect facts
quantum resource facts
```

Rule:

```text
body edit rechecks that body only unless public type/effect surface changes
```

### 8. Semantic Module Cache

Key:

```text
all type signature keys in module
all typed body keys in module
module graph key
visibility policy key
```

Output:

```text
CheckedFile
CheckedModule
SemanticModel slice
PublicApi slice
diagnostic replay bundle
```

### 9. HIR Cache

Key:

```text
typed body key
HIR lowering version
target-independent lowering policy
```

Output:

```text
HIR nodes
HIR function body
HIR module metadata
source map
```

### 10. MIR Cache

Key:

```text
HIR key
MIR lowering version
control-flow normalization policy
```

Output:

```text
MIR function body
CFG
locals
drop/resource plan
quantum operation plan
```

### 11. Optimization Cache

Key:

```text
MIR key
optimization profile
target-independent pass versions
```

Output:

```text
optimized MIR
pass summary
changed facts
```

Rule:

```text
debug profile can skip most optimization caches
release profile must cache pass outputs by function
```

### 12. Native Lowered Unit Cache

Key:

```text
optimized MIR function key
target triple
ABI key
data layout key
native backend version
```

Output:

```text
instruction selection result
register allocation result
frame layout
relocation metadata
object-unit preimage
```

### 13. Quantum Lowered Unit Cache

Key:

```text
quantum MIR/QMIR key
QPU target capability key
provider profile key
gate set key
transpile profile key
```

Output:

```text
QMIR/QTIR facts
QIR preimage
OpenQASM preimage
shot plan
noise/simulator plan
provider handoff metadata
```

### 14. Per-Unit Object Cache

Key:

```text
native lowered unit key
object format key
debug info policy
symbol mangling key
```

Output:

```text
.o unit
symbol table
relocations
debug map
```

Rule:

```text
one function body change should re-emit one object unit where possible
```

### 15. Link Cache

Key:

```text
ordered object unit keys
runtime/static archive keys
linker script key
target triple
link mode
```

Output:

```text
executable
static library
shared library
link map
diagnostics
```

Rule:

```text
if object keys unchanged, link is skipped
if only non-linked metadata changed, link is skipped
```

## Public API Versus Body Edits

Every source edit must classify into one of these buckets:

```text
no semantic change
body-only change
private signature change
public API change
module graph change
target/profile change
toolchain change
```

Impact:

```text
no semantic change:
  replay diagnostics and skip everything

body-only change:
  parse/check/lower/emit changed body

private signature change:
  recheck same module and affected private users

public API change:
  recheck import dependents

module graph change:
  rebuild graph and affected modules

target/profile change:
  reuse frontend, rerun target/backend layers

toolchain change:
  invalidate layers whose version changed
```

## Parallel Scheduler

qtlc1 must not pretend `-j` works. The scheduler must expose real parallelism:

```text
file lexing        parallel by file
parsing            parallel by file
signature scan     parallel by module
body checking      parallel by independent body
HIR/MIR lowering   parallel by function/body
native emit        parallel by object unit
quantum transpile  parallel by circuit/function
link               mostly serial, but inputs prepared in parallel
```

The progress UI must show active phase and counts:

```text
[LEX  ] files=120/500 workers=12 cache=380 hit
[PARSE] files=120/120 workers=12
[CHECK] bodies=900/1400 workers=12
[EMIT ] objects=44/200 workers=12
[LINK ] build/debug/app
```

Scheduler requirements:

```text
work stealing
dependency-ready tasks
critical-path priority
memory budget per worker
observable active-worker count
no global type-check lock
no global backend-emit lock
```

## Continuous Cache Writes

Cache products are written as soon as a unit completes:

```text
token cache after lexing each file
AST cache after parsing each file
typed body cache after each body
HIR/MIR cache after each body lower
object cache after each object unit
```

If the build is interrupted, next build resumes from completed units.

## Global And Local Cache Layout

User app cache:

```text
~/.cache/qtlc/
```

Final app outputs:

```text
./build/debug/
./build/release/
```

Toolchain cache:

```text
~/.quantumlang/qstore/
```

Toolchain install:

```text
~/.quantumlang/toolchains/<channel>/
/opt/quantumlang/toolchains/<channel>/
```

Do not mix user app cache with toolchain build cache.

## Dependency Source Avoidance

Normal app builds must not re-read dependency source when compiler-readable
metadata is available.

Input:

```text
sysroot.qidx
package qidx
*.qcap
```

Rule:

```text
valid qcap/qidx dependency:
  mmap metadata
  load symbols/types/generic IR by index
  do not scan dependency .qn files
```

This is required for stdlib, runtime, platform, plugins, and installed packages.

## No-Change Build Proof

Required proof command:

```text
qtlc build app
qtlc build app
```

Second run must prove:

```text
source_hit=true
tokens_reused=true
ast_reused=true
module_graph_reused=true
type_signatures_reused=true
typed_bodies_reused=true
hir_reused=true
mir_reused=true
objects_reused=true
link_skipped=true
```

## One-File-Change Proof

Required proof command:

```text
qtlc build app
edit src/foo.qn body only
qtlc build app
```

Second run must prove:

```text
changed_files=1
relexed_files=1
reparsed_files=1
rechecked_bodies=1 or affected_count
reused_modules=N-affected
reemitted_objects=1 or affected_count
link_ran=only_if_needed
```

## Public API Change Proof

Required proof:

```text
edit exported function signature
```

Build report must show:

```text
public_api_changed=true
affected_dependents=list
unaffected_modules_reused=true
```

## Cache Safety Rules

Never reuse cached products if:

```text
compiler version changed for that layer
target triple changed for backend layer
profile changed for optimization/backend layer
dependency API key changed
diagnostic policy changed
source hash mismatched
artifact schema version mismatched
```

Never silently fall back to stale artifacts. If reuse is rejected, report the
reason.

## Build Report Contract

Every build must emit a structured summary:

```text
files_total
files_changed
tokens_hit/miss
ast_hit/miss
signatures_hit/miss
typed_bodies_hit/miss
hir_hit/miss
mir_hit/miss
objects_hit/miss
link_hit/miss
workers_requested
workers_used
cache_reject_reasons
```

This is how users know the build system is really fast, not just quiet.

## Performance Proof Budgets

Initial budgets for a medium package:

```text
hot no-change build:
  under 200 ms where hardware allows

cold no-change build after warm cache:
  under 1 s where hardware allows

hot one-body edit:
  under 500 ms frontend plus affected backend unit where hardware allows

public API edit:
  proportional to real symbol dependents, not whole package
```

These are proof targets. If hardware or project size makes a target impossible,
the build report must still prove minimal work and explain the bottleneck.

## Implementation Order

Performance implementation must follow compiler slices:

```text
1. Stable ID model and query keys
2. BuildDatabase, ProductStore, DependencyGraph, DirtySet, CacheReport
3. Source fingerprint and line-map cache
4. Token cache
5. Green-tree AST cache
6. Import/export signature cache
7. Symbol-level dependency edges
8. Type signature cache
9. Typed body cache
10. HIR/MIR cache
11. Generic instantiation cache
12. Native/quantum lowered-unit cache
13. Object cache
14. Link cache
15. qcap/qidx dependency source-avoidance proof
16. qtlcd hot-build protocol
17. No-change build proof
18. One-file-change proof
19. Public API change proof
```

Do not wait for the full compiler to exist before designing cache keys. Each
slice must produce its cache key and replay contract when implemented.
