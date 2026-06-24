# qtlc1 Implementation Command Roadmap

Status: command ledger for completing qtlc1 as the qtlc2 builder

Rule: each command must leave qtlc1 in a qtlc0-checkable state. Do not merge
large compiler behavior into one command.

Primary goal: qtlc1 exists to build qtlc2. It is not the final toolchain and
must not drift into runtime, stdlib, package manager, plugin runtime, IDE, or
final language ecosystem work before the qtlc2 handoff.

Every command must answer:

```text
Does this make qtlc1 more capable of building qtlc2?
Does it preserve or improve fast incremental rebuild behavior?
Does it leave a qtlc0-checkable proof?
```

If the answer is no, the work belongs to qtlc2/qtlc3 or another roadmap.

The complete normal-project global-cache and query-driven implementation
sequence is locked in
[QTLC1_NORMAL_PROJECT_COMPILATION_IMPL_COMMANDS.md](QTLC1_NORMAL_PROJECT_COMPILATION_IMPL_COMMANDS.md).

The qtlc1/qtlc2/qtlc3 bytes and hash boundary is locked in
[QTLC_BYTES_HASH_STAGE_ROADMAP.md](QTLC_BYTES_HASH_STAGE_ROADMAP.md).

Current normal-project implementation position:

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

## qtlc2 Builder Contract

qtlc1 must become a small but serious compiler that can build qtlc2 source with
superfast rebuild behavior.

Required qtlc1 behavior:

```text
cold build qtlc2 from source
check qtlc2 module/import/export boundaries
type-check qtlc2 compiler source
lower qtlc2 checked source into HIR/MIR products
emit qtlc2 native object-unit products
link qtlc2 executable/package artifacts
emit qtlc2 .qi/.qcap/.qidx interface products
replay unchanged qtlc2 source/token/AST/type/HIR/MIR/object products
invalidate exactly the changed source/body/symbol dependents
avoid dependency source reads through qcap/qidx when possible
support qtlcd hot state for repeated qtlc2 edits
report cache hits/misses and changed products clearly
```

Forbidden before qtlc2 handoff:

```text
runtime implementation
stdlib implementation
HTTP/package implementation
plugin runtime implementation
package manager
LSP/formatter/docs generator
final async runtime
final advanced OOP ecosystem
```

## qtlc0 Prerequisite: Explicit Ownership Symbols

Before continuing declaration-arena and self-hosted frontend work, qtlc0 needs
the explicit ownership surface locked in
[QTLC0_EXPLICIT_OWNERSHIP_SYMBOLS_IMPL.md](QTLC0_EXPLICIT_OWNERSHIP_SYMBOLS_IMPL.md).

Required qtlc0 rule:

```text
value       read borrow/default for non-copy objects
&value      explicit read borrow
^value      explicit move/consume
&!value     mutable borrow
*T          mutable pointer target
*const T    read-only pointer target
...value    spread/variadic, already separate
```

This prevents qtlc1 declaration/frontend code from depending on hidden
aggregate moves. qtlc1 should remain clean: normal reads use `value`; only real
consuming APIs use `^value`.

## qtlc2 Proof Gates

The final qtlc1 gate is not just "qtlc1 builds". It must prove qtlc1 can build
qtlc2 with controlled incremental behavior.

Required proof sequence:

```text
qtlc0 check qtlang/compiler
qtlc0 build qtlang/compiler -> build/debug/qtlc1
build/debug/qtlc1 check qtlang/compiler
build/debug/qtlc1 build qtlang/compiler -> build/debug/qtlc2
build/debug/qtlc2 check qtlang/compiler
```

Required fast-build proof:

```text
no-change qtlc2 rebuild
  source/token/AST/type/HIR/MIR/object products replayed
  link skipped or reused when artifact identity is unchanged

one private body change
  changed body invalidated
  same public API dependents reused
  only affected HIR/MIR/object/link products rebuilt

one public API change
  exact exported symbol dependents invalidated
  unrelated modules/products reused

dependency-only build
  dependency qcap/qidx used
  dependency source avoided
```

## qtlc1 Superfast Check Plan

`qtlc1 check` must become a real fast compiler check, not only a correctness
proof. The current self-check behavior is intentionally visible but not yet
acceptable as the final qtlc1 builder path:

```text
observed: qtlang/compiler/build/debug/qtlc1 check qtlang/compiler
status: single-job check, full frontend/symbol/scope/type passes
time: about 500s for 625 files / about 157k typed nodes
progress issue: phase-local percentages reset from Scopes to Type records to
                Typechecking, which looks like a second check even though it is
                a later phase in the same check command
correctness blocker observed: unresolved member call 'this.byteSize' in
                              extend<T: ByteCodec<T>> T
```

The fast-check implementation order is locked:

```text
1. indexed hot-path lookups
2. real qtlc1 jobs plumbing
3. deterministic frontend file shards
4. deterministic typecheck shards
5. qtlc1 typed-product shard cache
6. qtlcd hot check/build state
7. global progress model
8. final qtlc1/qtlc2 magic-fast proof gates
```

Do not implement worker parallelism before indexed lookup tables. Parallelizing
linear scans only multiplies bad work and hides the actual compiler design
problem.

```text
POST-QTLC1-SUPERFAST-CHECK-001
  planned: replace linear TypeScopeTable hot lookups with deterministic indexes:
    scopeById
    parentByScope
    moduleScopeByHash
    ownerScopeBySymbol
  reason: scopeValue, parentOf, moduleScopeForHash, and ownerScopeFor currently
          scan scopeValues. These paths are used during scope construction,
          expression resolution, receiver lookup, and enclosing-owner lookup.
  constraints:
    keep nodeScopeValues[index] as the direct node-row lookup
    keep bindingLookup as the authoritative (ScopeId, name) binding index
    preserve deterministic scope order and existing scope ids
    no qtlc1 source weakening; if qtlc0 cannot lower the index shape, fix qtlc0
  proof target:
    qtlc0 check qtlang/compiler/typeSystem
    qtlc0 check qtlang/compiler
    qtlc0 build qtlang/compiler
    build/debug/qtlc1 check qtlang/compiler

POST-QTLC1-SUPERFAST-CHECK-002
  planned: make qtlc1 jobs real instead of cosmetic.
  required API change:
    frontendProductForPackage(target: string, jobs: i64)
    FrontendProduct.forPackage(target: string, jobs: i64)
    RealFrontend(target: string, jobs: i64)
    TypeEnvironment.forFrontend(frontend: FrontendProduct, jobs: i64)
  command behavior:
    build/debug/qtlc1 check -j10 qtlang/compiler
    build/debug/qtlc1 build -j10 qtlang/compiler
  constraints:
    HostArgs.jobs is already parsed; Session must pass args.jobs into frontend
    and typecheck construction
    jobs <= 1 must preserve the current deterministic serial behavior exactly
  proof target:
    qtlc1 check -j1 qtlang/compiler equals qtlc1 check qtlang/compiler
    qtlc1 check -j10 qtlang/compiler exits with the same diagnostics/result

POST-QTLC1-SUPERFAST-CHECK-003
  planned: deterministic parallel frontend file shards.
  design:
    each worker reads, lexes, and parses an assigned source file row
    worker output is FrontendFileShard(fileRow, fileId, lexProduct, astProduct,
                                      sourceFingerprint, diagnostics)
    the main thread merges shards strictly by source file row
  constraints:
    no shared mutation of PackageLexProduct or PackageAstProduct from workers
    all diagnostics and AST rows remain deterministic after merge
    module path/hash identity must come from the real source row
  proof target:
    qtlc1 check -j10 qtlang/compiler produces the same token, AST, declaration,
    and diagnostic counts as qtlc1 check -j1 qtlang/compiler

POST-QTLC1-SUPERFAST-CHECK-004
  planned: deterministic typecheck shards after symbols/scopes/candidates are
  immutable.
  design:
    build symbols, scopes, and candidate index once
    assign AST node ranges or file ranges to workers
    worker output is TypecheckShard(fileRow/range, typedNodeValues,
                                   local diagnostics, counters)
    merge shards by file row and node order
  constraints:
    no shared TypeRegistry mutation from workers
    expression resolution must only read symbols/scopes/candidates
    previous/next operand lookup must stay file-local and deterministic
  proof target:
    qtlc1 check -j10 qtlang/compiler has identical typed node count, type fact
    count, diagnostic count, and diagnostic messages as -j1

POST-QTLC1-SUPERFAST-CHECK-005
  planned: qtlc1-owned typed-product shard cache.
  products:
    source fingerprint shard
    token shard
    AST shard
    declaration shard
    scope/symbol summary shard
    typed-node shard
    diagnostic shard
  invalidation:
    unchanged private body reuses declaration/scope surfaces
    public API change invalidates exact exported symbol dependents
    import/export change invalidates affected module graph closure
  proof target:
    no-change qtlc1 check replays shards
    one private body edit rechecks only the changed body and affected local facts

POST-QTLC1-SUPERFAST-CHECK-006
  planned: qtlcd hot check/build state for qtlc1 and qtlc2.
  hot state:
    source table
    lex products
    AST products
    declaration table
    symbol table
    scope table and lookup indexes
    candidate index
    typed facts
    diagnostics
  behavior:
    repeated check avoids disk/source/package-wide rebuild when fingerprints
    and dependency keys prove reuse
  proof target:
    qtlcd no-change check returns from hot state
    qtlcd one-file private body edit touches only that file's typed shard and
    exact reverse dependents

POST-QTLC1-SUPERFAST-CHECK-007
  planned: global progress instead of phase-local percent reset.
  output shape:
    [Check] 62% phase=Typechecking local=28% file=...
  constraints:
    quiet mode prints only final result and diagnostics
    normal mode must not look like check restarted from zero
    verbose mode may print phase-local totals, jobs, tokens, symbols, scopes,
    cache hits, and changed shard counts
  proof target:
    qtlc1 check qtlang/compiler displays monotonic global progress

POST-QTLC1-SUPERFAST-CHECK-008
  planned: final superfast proof gates.
  required commands:
    qtlc0 check qtlang/compiler
    qtlc0 build qtlang/compiler
    build/debug/qtlc1 check -j10 qtlang/compiler
    build/debug/qtlc1 build -j10 qtlang/compiler
    build/debug/qtlc2 check -j10 qtlang/compiler
  required measurements:
    cold check time
    no-change check time
    one private body edit time
    one public API edit time
    cache/shard hit counts
    diagnostics equality against serial -j1
```

## Current Status

```text
POST-QTLC1-MAGIC-FAST-001
  implemented: stable ID model, query keys, and initial query data shells
  proof: qtlc0 check qtlang/compiler/query, qtlc0 check qtlang/compiler,
         qtlc0 build -j4 qtlang/compiler

POST-QTLC1-MAGIC-FAST-002
  implemented: BuildDatabase execution API, product lookup shape, validation
  contracts, query execution records, and cache report update API
  proof: qtlc0 check qtlang/compiler/query, qtlc0 check qtlang/compiler,
         qtlc0 build -j4 qtlang/compiler

POST-QTLC1-SOURCE-STYLE-002
  implemented: query data shapes use locked QuantumLang visibility style,
  no pub struct, no redundant public fields, no field defaults
  proof: qtlc0 check qtlang/compiler/query, qtlc0 check qtlang/compiler,
         qtlc0 build -j4 qtlang/compiler

POST-QTLC0-APP-BINARY-ARTIFACT-CONTRACT-001
  implemented: qtlc0 app targets with [library] still emit package-name
  executable by default; .qi surfaces move to interfaces/<importRoot>
  proof: qtlc0 dry-run qtlang/compiler artifact_mode=binary,
         qtlc0 build -j4 qtlang/compiler emits ELF build/debug/qtlc1,
         build/debug/qtlc1 exits 0

POST-QTLC1-MAGIC-FAST-003
  implemented: source fingerprint, token, and green-tree AST query product
  contracts with replayable product fingerprints
  proof: qtlc0 check qtlang/compiler/query, qtlc0 check qtlang/compiler,
         qtlc0 build -j4 qtlang/compiler emits ELF build/debug/qtlc1,
         build/debug/qtlc1 exits 0

POST-QTLC1-MAGIC-FAST-004
  implemented: export surface product, exported symbol records, and
  symbol-level dependency edge/set contracts
  proof: qtlc0 check qtlang/compiler/query, qtlc0 check qtlang/compiler,
         qtlc0 build -j4 qtlang/compiler emits ELF build/debug/qtlc1,
         build/debug/qtlc1 exits 0

POST-QTLC1-MAGIC-FAST-005
  implemented: type signature, typed body, and typed node query product
  contracts with replayable product fingerprints
  proof: qtlc0 check qtlang/compiler/query, qtlc0 check qtlang/compiler,
         qtlc0 build -j4 qtlang/compiler emits ELF build/debug/qtlc1,
         build/debug/qtlc1 exits 0

POST-QTLC1-BOOTSTRAP-003
  implemented: qtlc0 parses qtlc1 [selfHost] manifest policy and materializes
  the first qtlc2 stage artifact from the linked qtlc1 binary; qtlc1 source has
  a self-host build report model under buildSystem
  proof: qtlc0 build -j8 qtlang/compiler emits executable build/debug/qtlc1
         and executable build/debug/qtlc2; both binaries run qtlc1 check and
         report status ok

POST-QTLC1-RUNTIME-ARGV-001
  implemented: qtlc1 command dispatch reads process args through compiler-local
  qtlc1HostArgEquals/qtlc1HostArgOr intrinsics. qtlc0 lowers those intrinsics
  to small runtime ABI helpers QnOsArgEq/QnOsArgOr without adding runtime,
  stdlib, or plugin packages to qtlc1.
  proof: qtlc0 rebuilds, qtlc0 build -j8 qtlang/compiler emits executable
  qtlc1/qtlc2, ./build/debug/qtlc1 build qtlang/compiler prints qtlc1 build
  status ok, ./build/debug/qtlc1 check qtlang/compiler prints check status ok,
  ./build/debug/qtlc1 version prints qtlc1 compiler 0.1.0, and
  ./build/debug/qtlc2 build qtlang/compiler dispatches through the same argv
  path.

POST-QTLC0-NATIVE-BINARY-SKIP-BACKEND-IDENTITY-001
  implemented: checked-source metadata binary skip now requires matching native
  link executable metadata before it can skip source/lowering/link. The link
  metadata validates backend_cache_version, dependency key hash, executable
  content hash, and native executable bytes. Build stamps also carry
  backend_cache_version, so backend/runtime-only qtlc0 changes invalidate stale
  up-to-date decisions.
  proof: with existing qtlc1/qtlc2 outputs present, bumping backend cache
  version makes qtlc0 reject the early binary skip with
  native-link-metadata-backend-cache-version-mismatch, rebuild/relink qtlc1, and
  then the next qtlc0 build returns up-to-date in 00:00. qtlc1 and qtlc2 both
  still dispatch `build qtlang/compiler` correctly.

POST-QTLC0-NATIVE-PER-UNIT-MANIFEST-STALENESS-GATE-001
  implemented: partial native emit can only reuse a per-unit object manifest
  when every manifest unit semantic_hash matches the current lowered unit set.
  This prevents stale object manifests from linking old qtlc1 code after a
  source change.
  proof: a stale checkSnapshot.qn object manifest with semantic_hash
  b7abf4929f0ab019 was rejected, qtlc0 rebuilt qtlc1 from fresh lowering,
  and the rewritten manifest now records semantic_hash a37f21bb5314adce for
  qtlang/compiler/driver/checkSnapshot.qn. qtlc1 check qtlang/compiler,
  qtlc1 build qtlang/compiler, and qtlc2 check qtlang/compiler all exit 0.

POST-QTLC1-BOOTSTRAP-004
  implemented: qtlc1 build now gates the qtlc2 stage report on a real qtlc1
  source check and prints the remaining product state explicitly.
  current proof: qtlc1 build qtlang/compiler reports source check ok,
  artifact qtlc0-stage-materialized, deterministic products
  pending-real-qtlc1-builder, and status ok. qtlc2 check qtlang/compiler
  reports status ok.
  remaining: replace qtlc0-stage-materialized with qtlc1-owned deterministic
  source/lower/object/link products.

POST-QTLC1-MAGIC-FAST-006
  implemented: HIR body and MIR body query product contracts with replayable
  product fingerprints
  proof: qtlc0 check qtlang/compiler/query, qtlc0 check qtlang/compiler,
         qtlc0 build -j4 qtlang/compiler emits ELF build/debug/qtlc1,
         build/debug/qtlc1 exits 0

POST-QTLC1-MAGIC-FAST-007
  implemented: generic argument records and generic instance query product
  contract with replayable product fingerprint
  proof: qtlc0 check qtlang/compiler/query, qtlc0 check qtlang/compiler,
         qtlc0 build -j4 qtlang/compiler emits ELF build/debug/qtlc1,
         build/debug/qtlc1 exits 0

POST-QTLC1-MAGIC-FAST-008
  implemented: native object product, link input, and linked target product
  contracts with target/profile cache identity
  proof: qtlc0 check qtlang/compiler/query, qtlc0 check qtlang/compiler,
         qtlc0 build -j4 qtlang/compiler emits ELF build/debug/qtlc1,
         build/debug/qtlc1 exits 0

POST-QTLC1-MAGIC-FAST-009
  implemented: qcap/qidx mmap handle contracts and dependency source
  avoidance proof contract
  proof: qtlc0 check qtlang/compiler/interface, qtlc0 check qtlang/compiler,
         qtlc0 build -j4 qtlang/compiler emits ELF build/debug/qtlc1,
         build/debug/qtlc1 exits 0

POST-QTLC1-MAGIC-FAST-010
  implemented: qtlcd hot-build protocol, request, response, and hot-state
  readiness contracts
  proof: qtlc0 check qtlang/compiler/buildSystem, qtlc0 check qtlang/compiler,
         qtlc0 build -j4 qtlang/compiler emits ELF build/debug/qtlc1,
         build/debug/qtlc1 exits 0

POST-QTLC1-MAGIC-FAST-011
  implemented: magic-fast proof scenario budgets, proof result, and
  changed-product report contracts
  proof: qtlc0 check qtlang/compiler/buildSystem, qtlc0 check qtlang/compiler,
         qtlc0 build -j4 qtlang/compiler emits ELF build/debug/qtlc1,
         build/debug/qtlc1 exits 0

POST-QTLC1-INCREMENTAL-QUERY-ENGINE-001
  implemented: qtlc1 incremental query planning model for no-change,
  private-body-change, public-API-change, and target-change rebuilds; proof
  requires unaffected source/token/type/body/native-object products to stay
  green where safe; qtlc1 artifact policy explicitly forbids generated .c,
  .cpp, and header files during build.
  proof: qtlc0 check qtlang/compiler/query, qtlc0 check
         qtlang/compiler/buildSystem, qtlc0 check qtlang/compiler/tests,
         qtlc0 check qtlang/compiler, qtlc0 build -j8 qtlang/compiler,
         qtlc0 build -j8 qtlang/compiler/tests, and tests binary exits 0.
  style note: ArtifactPolicy uses the auto-constructor from
  `pub ArtifactPolicy { ... }`; call `ArtifactPolicy()` directly.
  Do not add a duplicate explicit constructor or wrapper function.
```

## Quantum Source Style Cleanup

```text
POST-QTLC0-QUANTUM-FN-SUGAR-001
  verified: qtlc0 parses and builds name(args) without fn

POST-QTLC1-SOURCE-STYLE-001
  implemented: qtlc1 compiler source uses fn-free function declarations

POST-QTLC1-SOURCE-STYLE-002
  implemented: qtlc1 query data shapes use pub Type fields and private
  override style instead of pub struct/public field syntax

POST-QTLC1-SCOPE-STYLE-LOCK-001
  implemented: full qtlc1 implementation scope lock, qtlc0/qtlc1/qtlc2/qtlc3
  stage split, mandatory this/This style, no-self rule, compiler-only qtlc1
  boundary, Vec/Array/record/Map/Option literal rules, and literal scope rules

POST-QTLC1-CORE-TYPE-MEMORY-OPERATOR-SCOPE-001
  implemented: qtlc0-checked primitive type list, qtlc1 core memory fact scope,
  operator scope, compatibility aliases, and qtlc1/non-qtlc1 implementation
  boundary for primitive/runtime/stdlib surfaces

POST-QTLC1-CONTROL-FLOW-LOOP-SCOPE-001
  implemented: qtlc0-checked normal control-flow, loop, parallel loop, domain
  loop, and quantum qif/qfor/qwhile/qloop scope with qtlc1/qtlc2 boundaries

POST-QTLC1-PRIMITIVE-CATALOG-CLEANUP-001
  implemented: primitive/core/alias/layout catalog split, PrimitiveTypeFact,
  primitive ABI return facts, and builtin catalog facade cleanup
  proof: qtlc0 check qtlang/compiler/typeSystem, qtlc0 check qtlang/compiler

POST-QTLC1-TYPE-CATALOG-PROOF-001
  implemented: type catalog proof source for byte primitive, text/bytes view
  core types, int/uint-only aliases, old-name rejection, and matrix square
  structural normalization
  proof: qtlc0 check qtlang/compiler/typeSystem, qtlc0 check qtlang/compiler

POST-QTLC1-CORE-002
  started: literal typing policy/fact/proof scaffold for bool, integer, float,
  string, bytes, byte, Option, record, array, and map literals
  added: target-typed literal resolution facts for [], none, normal value
  lifting into T?, anonymous records, quoted-key maps, byte literals, and
  string literals
  locked: optional values use target-typed lifting (`let name: string? =
  "Ali"`) and typed absence (`let missing: string? = none`); untyped none
  and `let value: string = none` are rejected
  corrected: [] maps to Vec literal rules; fixed arrays are separate
  target-typed FixedArray literals
  added: literal diagnostic facts for none without Option target, unclear [],
  unclear map/object literal, duplicate record fields, map key mismatch, byte
  literal overflow, and integer literal overflow
  added: syntax/AST bridge facts mapping literal token/expression forms to
  TypeLiteralKind, target resolution, and literal diagnostics
  reference update: literal target compatibility matrix now lives in
  QTLC1_CORE_TYPE_MEMORY_OPERATOR_SCOPE.md and locks explicit future literal
  families for qubit/qreg/bit/bits/qfunc, nullptr/nullref, and fn(...) => ...
  without making normal scalar literals silently target quantum/function/
  pointer/reference types
  proof: qtlc0 check qtlang/compiler/typeSystem, qtlc0 check qtlang/compiler

POST-QTLC1-CORE-009
  planned: add explicit quantum/function/pointer/reference literal target
  skeleton after core scalar/aggregate typing is stable:
  qubit |0>, qreg |0000>, bit 1, bits 0b..., qfunc { ... }, nullptr,
  nullref, and fn(...) => ...
  boundary: design/type facts only in qtlc1; execution/runtime/lowering is
  qtlc2/qtlc3 work unless qtlc1 compiler source requires a minimal fact

POST-QTLC1-CORE-010
  current: .. half-open range, ..= inclusive range, ** power, ++/-- mutation,
  and => as syntax-level expression/lambda/view body operator
  deferred: ?? none coalescing
  not planned: ?: ternary, ===, %%
  boundary: add syntax/type facts first; lowering/constant folding only when
  qtlc1 source needs it or qtlc2/qtlc3 takes over richer semantics

POST-QTLC1-TYPESYSTEM-OPERATOR-CATALOG-005
  implemented: builtin operator catalog facts now classify operator symbol,
  family, precedence label, fixed result, target-gated result, mutation, and
  validity. Expression type checking routes binary/unary/assignment/access/
  call/index/cast/propagate through BuiltinOperatorFact.fromSymbol instead of
  scattering string checks.
  includes: catalog entries for .., ..=, **, ++/--, and => as syntax body
  facts; ?? and === remain invalid until explicitly implemented.
  proof: qtlc0 check qtlang/compiler/typeSystem, qtlc0 check qtlang/compiler

POST-QTLC1-CORE-003
  started: expression typing rule scaffold for literals, optional target
  lifting, none rejection, record/Vec/FixedArray/Map literals, void returns,
  this/This resolution, and basic binary operator result facts
  proof target: qtlc0 check qtlang/compiler/typeSystem, qtlc0 check
  qtlang/compiler

POST-QTLC1-CORE-004
  started: function/body typing rule scaffold for constructor return default
  to This, expression-body return inference, declared return enforcement, void
  block bodies, non-void explicit return requirements, and rejection of unclear
  block-body return inference
  proof target: qtlc0 check qtlang/compiler/typeSystem, qtlc0 check
  qtlang/compiler

POST-QTLC1-CORE-TEXT-API-001
  added: qtlc1-needed core text/bytes/char/byte method facts for compiler
  source: string byteLength/length/isEmpty/startsWith/endsWith/contains/
  indexOf/lastIndexOf/slice/substring/trim/trimStart/trimEnd/replace/toBytes,
  bytes length/isEmpty/byteAt/slice/toString, char lexer predicates, and byte
  ASCII lexer predicates
  proof target: qtlc0 check qtlang/compiler/typeSystem, qtlc0 check
  qtlang/compiler
```

## Scope Lock

```text
POST-QTLC1-SCOPE-STYLE-LOCK-001
  add full qtlc1 scope and Quantum style lock doc [done]

POST-QTLC1-CORE-TYPE-MEMORY-OPERATOR-SCOPE-001
  add qtlc0 primitive audit and qtlc1 core type/memory/operator scope doc [done]

POST-QTLC1-CONTROL-FLOW-LOOP-SCOPE-001
  add qtlc0 control-flow audit and qtlc1 loop/quantum-control scope doc [done]
```

## Foundation And Magic-Fast Build Core

```text
POST-QTLC1-MAGIC-FAST-001
  define stable ID model and query keys [done]

POST-QTLC1-MAGIC-FAST-002
  add BuildDatabase execution API, product lookup, and validation contracts [done]

POST-QTLC1-MAGIC-FAST-003
  add source fingerprint, token, and green-tree AST query products [done]

POST-QTLC1-MAGIC-FAST-004
  add export surface and symbol-level dependency edges [done]

POST-QTLC1-MAGIC-FAST-005
  add type signature and typed body query products [done]

POST-QTLC1-MAGIC-FAST-006
  add HIR/MIR body query products [done]

POST-QTLC1-MAGIC-FAST-007
  add generic instantiation cache [done]

POST-QTLC1-MAGIC-FAST-008
  add native object-unit cache and link cache [done]

POST-QTLC1-MAGIC-FAST-009
  prove qcap/qidx mmap dependency source avoidance [done]

POST-QTLC1-MAGIC-FAST-010
  add qtlcd hot-build protocol skeleton [done]

POST-QTLC1-MAGIC-FAST-011
  add no-change, one-body-change, public-API-change proof gates [done]
```

## Frontend

```text
POST-QTLC1-FRONTEND-001
  added: qtlc1 token/import rule scaffold for Quantum-style imports:
  `import std::io::{println}`, `import std::io`, `import std::io as console`;
  import/module paths use `::`, calls/access use `.`, dot imports and `::`
  calls are rejected
  added: print-family prelude facts; `print`, `println`, `eprint`,
  `eprintln`, and `debug` need no import, while advanced IO names still
  require an explicit module import
  proof target: qtlc0 check qtlang/compiler/syntax, qtlc0 check
  qtlang/compiler

POST-QTLC1-FRONTEND-002
  added: source text/file/span/line-map scaffold with file identity,
  content fingerprints, source spans carrying file id, byte-offset-based
  columns, late display-column computation, diagnostic label span contract,
  and source fingerprint cache-key rule
  proof target: qtlc0 check qtlang/compiler/source, qtlc0 check
  qtlang/compiler

POST-QTLC1-FRONTEND-003
  added: diagnostic code/category facts, labels carrying source spans,
  diagnostics with primary labels and severity, deterministic diagnostic bags,
  reporter color policy, and qtlc1 diagnostic policy proof
  proof target: qtlc0 check qtlang/compiler/diagnostics, qtlc0 check
  qtlang/compiler

POST-QTLC1-FRONTEND-004
  added: token kind/role facts, syntax tokens carrying source span and trivia
  counts, trivia channel/span facts, literal token/default-type facts, and
  token shape proof
  proof target: qtlc0 check qtlang/compiler/syntax, qtlc0 check
  qtlang/compiler

POST-QTLC1-FRONTEND-005
  added: scanner cursor/source tracking, lexer/token classification facts,
  number and string literal classifier facts, line-comment trivia facts,
  operator recognition for `::`, `.`, `=>`, `?`, `=`, and import-shape proof
  for selected, default module, and aliased module imports
  proof target: qtlc0 check qtlang/compiler/syntax, qtlc0 check
  qtlang/compiler

POST-QTLC1-FRONTEND-006
  added: green AST node model with AstNodeKind, stable node id from file/span/
  kind, child-count facts, item/expr/stmt/pattern/typeRef kind wrappers, and
  deterministic AST shape proof
  proof target: qtlc0 check qtlang/compiler/syntax, qtlc0 check
  qtlang/compiler

POST-QTLC1-FRONTEND-007
  added: parser shape facts for module roots, import/export/function items,
  let/return/block statements, literal/call/record expressions, name/optional/
  generic types, precedence facts, recovery sync facts, and parser proof
  proof target: qtlc0 check qtlang/compiler/syntax, qtlc0 check
  qtlang/compiler

POST-QTLC1-FRONTEND-008
  added: parser recovery sync sets, expected-token replay diagnostics,
  missing-token placeholders, deterministic diagnostic order facts, recovery
  continuation facts, and parser recovery proof
  proof target: qtlc0 check qtlang/compiler/syntax, qtlc0 check
  qtlang/compiler

POST-QTLC1-FRONTEND-REAL-SOURCE-001
  added: qtlc1 host-backed real main-source check; qtlc1 now validates the
  compiled package has a real main.qn, non-empty source text, a main function,
  Quantum-style import paths, and print-family prelude usage without needing
  a qtlc1 filesystem runtime
  proof target: qtlc0 check qtlang/compiler/source, qtlc0 check
  qtlang/compiler, qtlc0 build qtlang/compiler, build/debug/qtlc1 check
  qtlang/compiler

POST-QTLC1-FRONTEND-REAL-LEXER-001
  added: qtlc1 host-backed real main-source lexer check; qtlc1 now proves the
  actual main.qn has non-zero lexer tokens and recognizes import, module-path
  `::`, and print-family call tokens through qtlc0-folded facts while the real
  qtlc1 filesystem/lexer runtime is still being built
  proof target: qtlc0 check qtlang/compiler/syntax, qtlc0 check
  qtlang/compiler, qtlc0 build qtlang/compiler, build/debug/qtlc1 check
  qtlang/compiler

POST-QTLC1-FRONTEND-REAL-PARSER-001
  added: qtlc1 host-backed real main-source parser-structure check; qtlc1 now
  proves the actual main.qn has a module root, import item, function item,
  return statement, and call expression through qtlc0-folded facts while the
  real qtlc1 parser runtime is still being built
  proof target: qtlc0 check qtlang/compiler/syntax, qtlc0 check
  qtlang/compiler, qtlc0 build qtlang/compiler, build/debug/qtlc1 check
  qtlang/compiler

POST-QTLC1-FRONTEND-SOURCE-READER-001
  added: qtlc1-owned source reader request/product/proof contracts carrying
  target, package-relative path, SourceFile, SourceText, deterministic cache
  keys, and text fingerprint identity; this is the first frontend product
  layer that will replace host-backed source facts
  proof target: qtlc0 check qtlang/compiler/source, qtlc0 check
  qtlang/compiler, qtlc0 build qtlang/compiler, build/debug/qtlc1 check
  qtlang/compiler
```

## Control Flow And Loops

```text
POST-QTLC1-CFLOW-001
  added: normal control-flow kind/facts for if/if-else/match/return/require/
  expr/block, AST statement bridge, require condition diagnostics, and proof
  for branch region creation, return path termination, require failure path,
  and statement mapping
  proof target: qtlc0 check qtlang/compiler/controlFlow, qtlc0 check
  qtlang/compiler

POST-QTLC1-CFLOW-002
  added: loop/while/while-let/for/range loop facts, labeled break/continue
  target resolution, unresolved transfer diagnostics, loop result break-value
  facts, AST loop bridge, explicit qtlc1 loop syntax facts for loop/while/
  while-let/for-in/for-range/break/continue/labeled break/labeled continue,
  and loop control-flow proof
  proof target: qtlc0 check qtlang/compiler/controlFlow, qtlc0 check
  qtlang/compiler

POST-QTLC1-CFLOW-003
  added: deterministic CFG facts with entry/exit nodes, control-flow edges,
  regions with entry/exit nodes, reachability/unreachable diagnostics,
  definite-return/missing-return diagnostics, loop-result compatibility, and
  CFG proof
  proof target: qtlc0 check qtlang/compiler/controlFlow, qtlc0 check
  qtlang/compiler

POST-QTLC1-CFLOW-004
  deferred: par-for/shard-for are not qtlc1-required; qtlc1 records a
  deferred diagnostic/policy fact and leaves parallel lowering for qtlc2/qtlc3
  proof target: qtlc0 check qtlang/compiler/controlFlow, qtlc0 check
  qtlang/compiler

POST-QTLC1-CFLOW-005
  added: qtlc1-supported normal/loop control-flow lowering facts into HIR/MIR
  product names, deterministic HIR/MIR product proof, and explicit exclusion
  of parallel lowering from qtlc1
  proof target: qtlc0 check qtlang/compiler/controlFlow, qtlc0 check
  qtlang/compiler
```

## Module And Names

```text
POST-QTLC1-MODULE-001
  implement quantum.toml and module.toml manifest facts

POST-QTLC1-MODULE-002
  implement mod.qn module facade import/export facts

POST-QTLC1-MODULE-003
  implement package graph and module graph construction

POST-QTLC1-MODULE-004
  implement visibility and module-boundary diagnostics

POST-QTLC1-RESOLVE-001
  implement scopes and symbol tables

POST-QTLC1-RESOLVE-002
  resolve imports, exports, local names, and qualified paths

POST-QTLC1-RESOLVE-003
  implement overload candidate sets

POST-QTLC1-RESOLVE-004
  emit stable unresolved-name and private-visibility diagnostics
```

## Types And Semantics

```text
POST-QTLC1-CORE-001
  add builtin primitive catalog and compatibility alias table

POST-QTLC1-CORE-002
  add numeric, string, bytes, bool, void, option, and record literal typing

POST-QTLC1-CORE-003
  add operator precedence and core operator typing rules

POST-QTLC1-CORE-005
  add pointer, reference, array, slice, and nullability facts

POST-QTLC1-CORE-006
  add target data-layout and ABI return facts

POST-QTLC1-CORE-006
  add Option/Result carrier facts and `?` propagation typing

POST-QTLC1-CORE-007
  add Vec/Map/Set/record literal target-typing facts

POST-QTLC1-CORE-008
  add quantum core type metadata facts for qubit/qreg/qresult/qfunc/qany

POST-QTLC1-TYPE-001
  implement builtin type catalog and StableTypeId bridge

POST-QTLC1-TYPE-002
  implement data shape, field, constructor, and method signatures

POST-QTLC1-TYPE-003
  implement function signatures and expression type checking

POST-QTLC1-TYPE-004
  implement minimal generics needed by qtlc1

POST-QTLC1-TYPE-005
  implement effect and capability facts

POST-QTLC1-SEMANTIC-001
  produce CheckedFile and CheckedModule products

POST-QTLC1-SEMANTIC-002
  produce PublicApi and dependency fingerprints

POST-QTLC1-SEMANTIC-003
  replay semantic diagnostics from cached query products
```

## Quantum Language Core

```text
POST-QTLC1-QUANTUM-SCOPE-AUDIT-001
  added: checked qtlc1 quantum scope audit separating quantum control from
  normal controlFlow, covering qubit/qreg/qresult/qfunc/qany, resource/no-clone/
  lifetime, effects, qif/qfor/qwhile/qloop, verifiers, and QMIR/QTIR lowering
  proof target: qtlc0 check qtlang/compiler/quantum, qtlc0 check
  qtlang/compiler

POST-QTLC1-QUANTUM-001
  added: qubit/qreg/qresult/qfunc/qany metadata facts for linear/no-clone
  qubits, qreg qubit collections and qfor iteration, measurement qresult
  carrier, target-gated qfunc, qany ownership preservation, and metadata proof
  proof target: qtlc0 check qtlang/compiler/quantum, qtlc0 check
  qtlang/compiler

POST-QTLC1-QUANTUM-002
  added: quantum ownership/no-clone/lifetime/reset/release/allocation facts,
  diagnostics for ownership violation, clone rejection, lifetime leak, missing
  reset, and release-before-reset, plus quantum resource proof
  proof target: qtlc0 check qtlang/compiler/quantum, qtlc0 check
  qtlang/compiler

POST-QTLC1-QUANTUM-003
  added: quantum effect/purity/measurement/entanglement facts, gate/gate-set/
  intrinsic/operation/measurement/barrier/control facts, pure-qfunc measurement
  rejection, and quantum effect proof
  proof target: qtlc0 check qtlang/compiler/quantum, qtlc0 check
  qtlang/compiler

POST-QTLC1-QUANTUM-004
  added: quantum circuit/builder/topology/schedule/parameter/control-flow
  profile facts, deterministic circuit builder, topology-aware schedule,
  bounded profile facts, and quantum circuit proof
  proof target: qtlc0 check qtlang/compiler/quantum, qtlc0 check
  qtlang/compiler

POST-QTLC1-QUANTUM-005
  added: quantum resource/effect/target verifier facts, aggregate verifier,
  diagnostics for rejected resources/effects/targets, and verifier proof
  proof target: qtlc0 check qtlang/compiler/quantum, qtlc0 check
  qtlang/compiler

POST-QTLC1-QUANTUM-006
  added: QMIR/QTIR product facts, quantum AST/HIR/MIR lowering facts,
  deterministic lowering proof, and backend-ready quantum MIR handoff fact
  proof target: qtlc0 check qtlang/compiler/quantum, qtlc0 check
  qtlang/compiler

POST-QTLC1-QCONTROL-001
  added: qif/qfor/qwhile/qloop keyword token facts, AST statement kinds,
  parser statement facts, quantum control-flow facts, and syntax proof
  proof target: qtlc0 check qtlang/compiler/syntax, qtlc0 check
  qtlang/compiler/quantum, qtlc0 check qtlang/compiler

POST-QTLC1-QCONTROL-002
  added: qif path-sensitive branch facts, deterministic branch merge fact,
  measurement-condition tracking, no-clone preservation, reset/release balance,
  and diagnostic proof for rejected mismatched branch resource flow
  proof target: qtlc0 check qtlang/compiler/quantum, qtlc0 check
  qtlang/compiler

POST-QTLC1-QCONTROL-003
  added: qfor qreg-only iteration fact, per-iteration borrow fact,
  no-clone/no-leak loop-body fact, and diagnostic proof for non-qreg
  iteration and leaking loop bodies
  proof target: qtlc0 check qtlang/compiler/quantum, qtlc0 check
  qtlang/compiler

POST-QTLC1-QCONTROL-004
  added: qwhile/qloop restricted-profile fact, loop bound fact,
  deterministic-exit/runtime-guard proof, and diagnostics for unsupported
  profiles and unbounded quantum loops
  proof target: qtlc0 check qtlang/compiler/quantum, qtlc0 check
  qtlang/compiler

POST-QTLC1-QCONTROL-005
  added: quantum control-flow lowering fact for qif/qfor/qwhile/qloop,
  deterministic QMIR/QTIR product handoff, backend-ready MIR lowering proof,
  and stable diagnostic for rejected quantum control lowering
  proof target: qtlc0 check qtlang/compiler/quantum, qtlc0 check
  qtlang/compiler
  lower quantum control flow into QMIR/QTIR products
```

## IR, Target, And Backend

```text
POST-QTLC1-IR-001
  added: HIR item/function/block/stmt/expr/type/control-flow product shapes,
  MIR body/block/value/instruction/terminator product shapes, HIR/MIR shape
  proofs, and stable IR product boundary proof
  proof target: qtlc0 check qtlang/compiler/hir, qtlc0 check
  qtlang/compiler/mir, qtlc0 check qtlang/compiler

POST-QTLC1-IR-002
  added: checked source to HIR lowering fact, semantic model attachment,
  stable HIR product naming, and proof that checked control-flow survives
  into HIR
  proof target: qtlc0 check qtlang/compiler/hir, qtlc0 check qtlang/compiler

POST-QTLC1-IR-003
  added: HIR function/control-flow to MIR body lowering fact, CFG product,
  return/branch/require terminator mapping, deterministic MIR product proof,
  and stable diagnostic for rejected IR lowering
  proof target: qtlc0 check qtlang/compiler/mir, qtlc0 check qtlang/compiler

POST-QTLC1-IR-004
  added: debug-safe optimization pass manager, literals-only const fold,
  no-op inline pass, unreachable-only dead code pass, analysis-only escape
  pass, stable debug pass product plan, and optimizer proof
  proof target: qtlc0 check qtlang/compiler/optimize, qtlc0 check
  qtlang/compiler

POST-QTLC1-TARGET-001
  added: target triple, target feature, feature-set, data-layout,
  calling-convention, ABI, target-machine facts, and x86_64-linux proof
  proof target: qtlc0 check qtlang/compiler/machine, qtlc0 check
  qtlang/compiler

POST-QTLC1-TARGET-002
  added: Linux/Windows/macOS/no-OS platform facts, embedded memory map,
  interrupt vector, startup, linker script, device/board profile facts, and
  hosted + embedded platform proof
  proof target: qtlc0 check qtlang/compiler/platform, qtlc0 check
  qtlang/compiler

POST-QTLC1-BACKEND-001
  added: native MIR lowering, instruction selection, register allocation,
  frame layout, backend readiness facts, and native plan proof over MIR +
  target-machine products
  proof target: qtlc0 check qtlang/compiler/backend/native, qtlc0 check
  qtlang/compiler

POST-QTLC1-BACKEND-002
  added: native object unit key, object unit, object product, object emission
  counts, cache reuse/miss facts, and proof for unchanged MIR reuse, changed
  body one-unit rebuild, and target-change reuse rejection
  proof target: qtlc0 check qtlang/compiler/backend/native, qtlc0 check
  qtlang/compiler

POST-QTLC1-BACKEND-003
  added: link input set identity, link plan, link product reuse/miss contract,
  clean native artifact layout, and proof for no-change link reuse plus
  changed-object relink
  proof target: qtlc0 check qtlang/compiler/backend/native, qtlc0 check
  qtlang/compiler

POST-QTLC1-BACKEND-004
  added: quantum backend/provider/target profile facts, deterministic QIR and
  OpenQASM emitter facts, transpile/shot/noise/simulator plans, quantum
  artifact products, and backend proof for local simulator target
  proof target: qtlc0 check qtlang/compiler/backend/quantum, qtlc0 check
  qtlang/compiler
```

## Interface And Self-Host

```text
POST-QTLC1-INTERFACE-001
  added: human-readable .qi declaration, .qi file, interface layout,
  deterministic writer, API surface facts, and proof for interfaces/<importRoot>
  output naming
  proof target: qtlc0 check qtlang/compiler/interface, qtlc0 check
  qtlang/compiler

POST-QTLC1-INTERFACE-002
  added: qcap writer, section, product, content hash/count contract, and
  proof that emitted capsules can be memory-mapped through QcapHandle
  proof target: qtlc0 check qtlang/compiler/interface, qtlc0 check
  qtlang/compiler

POST-QTLC1-INTERFACE-003
  added: qidx writer, index entry, index product, memory-mapped QidxHandle
  proof, and fast lookup mapping from module/symbol to capsule offset
  proof target: qtlc0 check qtlang/compiler/interface, qtlc0 check
  qtlang/compiler

POST-QTLC1-BUILD-EMIT-CACHE-HANG-001
  fixed: qtlc0 native emit for qtlc1 no longer restores/writes the huge
  monolithic native product cache when per-unit lowered shards are available
  added: bounded direct OOP proof scan, bounded large-package production
  contract, split emit heartbeat labels, and monolithic-cache skip proof line
  proof target: cmake --build qtlc/build --target qtlc -j8; qtlc0 build -j8
  qtlang/compiler

POST-QTLC1-NATIVE-OBJECT-RESTORE-HIT-001
  superseded correction: native per-unit object cache keys and manifest object
  paths must attach by full lowered-unit index for every emitted unit, not by a
  compact body-unit index. Bodyless module shards still produce stable object
  slots, and skipping them shifts provider objects onto the wrong .o path.
  fixed: per-unit object action keys no longer include full qtlc0 executable
  identity; they use a stable native object ABI cache identity plus backend
  cache version/source/body/runtime inputs
  superseded correction: manifest fallback action hashes are disabled for this
  path because the old positional manifest poisoned global object entries.
  native-object-unit-path-v2 forces a fresh object write under the corrected
  unit-index mapping.
  proof target: qtlc0 build -j8 qtlang/qtlc0_test_build after qtlc0 driver
  rebuild reports fresh_objects=1, restored_objects=14, compiled_objects=0;
  immediate no-change rebuild reports status=up-to-date
  proof target: qtlc0 build -j8 qtlang/compiler after the v2 identity bump
  reports fallback_keys=0, compiled_objects=422 once, writes
  native-backend.per-unit-object-cache valid=true units=422
  split_ready_units=422, then immediate no-change rebuild reports
  status=up-to-date in 00:00

POST-QTLC1-LINK-EXECUTABLE-CACHE-HIT-001
  fixed: native executable link cache delivery now runs for direct binary
  output paths and binary-only cache hits clear object compile invocations
  added: visible native link executable cache probe with action/hit/delivery
  reason
  proof target: qtlc0 build -j8 qtlang/compiler reports action_hit=true,
  delivered=true, linked=0, link_skipped=1, fresh_objects=0, compiled_objects=0

POST-QTLC1-NATIVE-BINARY-EMIT-SKIP-001
  fixed: native binary builds can now skip lowering, native emit, object
  materialization, and link when checked-source hydrate, native unit cache, and
  lowered-unit shard materializer all prove every backend unit is unchanged
  proof target: qtlc0 build -j8 qtlang/compiler reports
  native binary emit skipped reason=native-unit-and-lowered-cache-hit and
  status=up-to-date before LOWER/EMIT

POST-QTLC1-NATIVE-BINARY-FRONTEND-SKIP-001
  fixed: native binary builds can now skip checked-source hydrate/typecheck when
  loaded source/manifest metadata key matches and the binary output exists,
  even after qtlc0 itself was rebuilt
  proof target: qtlc0 build -j8 qtlang/compiler reports
  checked-source metadata binary skip accepted reason=native-binary-source-key-match
  and exits in the precheck path before checked-source hydrate

POST-QTLC1-FRONTEND-STATE-EXECUTION-ENABLE-001
  fixed: source-stable full frontend-state execution is enabled by default for
  native builds after qtlc0 compiler-identity changes. If the source/module
  hydrate plan proves parse=0, check=0, blocked_modules=0, and live
  frontend/token/AST/resolve/type/semantic/module-graph artifacts validate,
  qtlc0 materializes a live CheckedSourceSet and skips the checked-source
  builder instead of running CHECK as a dry-run proof.
  proof target: qtlc0 build -j8 qtlang/qtlc0_test_build reports
  checked-source full-typed-set applied=true, parse=0, check=0, hydrate=15,
  checked-source builder skipped, backend product restored, fresh_objects=1,
  restored_objects=14, compiled_objects=0.
  proof target: qtlc0 build -j8 qtlang/compiler reports checked-source
  full-typed-set applied=true, parse=0, check=0, hydrate=422,
  checked-source builder skipped, native lowered-unit cache partial
  cached=422 unchanged=413 changed=9, backend-unit-lowering units=9
  restored_units=413, then after the corrected v2 object identity refresh
  compiled_objects=422 once and build/debug/qtlc1 is rebuilt successfully.
  remaining blocker: after the corrected manifest is populated, the next
  source-delta proof should demonstrate partial native emit and partial link
  from the 422-unit manifest without a full object refresh.

POST-QTLC1-INCREMENTAL-ONE-FILE-CHANGE-PROOF-001
  proved: a one-file qtlc1 source change does not take the native binary
  metadata fast path; checked-source reports source-key-changed, hydrate=398,
  recheck=1, recheck_modules=main, native unit cache reports changed=1, and
  lowering schedules one executable unit
  blocker found: artifact materialization still walks/restores many object
  inputs and relinks after a one-file executable change; next work must make
  native emit/link operate from the changed object plus cached object manifest
  without revalidating all 399 object inputs

POST-QTLC1-INCREMENTAL-CHANGED-OBJECT-MATERIALIZATION-001
  fixed: native partial-delta builds now filter compile/materialization
  invocations down to changed lowered-unit object paths while preserving the
  full linker input object list from the link plan
  proof target: one-file qtlc1 main.qn edit reports
  native changed-object materialization plan original_invocations=399
  filtered_invocations=1 changed_units=1; object reuse reports
  compiled_objects=1 for the changed edit, and after restore link cache reports
  linked=0 link_skipped=1 compiled_objects=0
  remaining blocker: native emit still rebuilds package-level source-map and
  production-contract metadata for all 399 units before object materialization

POST-QTLC1-INCREMENTAL-PARTIAL-NATIVE-EMIT-002
  fixed: native package inputs now carry stable original unit_index so
  changed-only emit keeps the same qtlc_artifacts/*.native.units/<index> path
  fixed: partial native emit keeps all 399 modules as metadata inputs but marks
  only changed units emit_bodies=true, so runtime-support discovery and ABI
  metadata remain complete while assembly/source-map output is bounded to the
  changed unit
  fixed: partial native link recovers unchanged object inputs from
  native-backend.per-unit-object-cache and appends the hidden runtime archive
  after recovered objects so static archive symbol resolution remains correct
  proof target: restoring qtlang/compiler/main.qn after a one-file edit reports
  changed_inputs=1 metadata_inputs=399, appended_from_manifest=287,
  compile_invocations=1, emitted=2, compiled_objects=0, linked=0,
  link_skipped=1; no-change build then reports status=up-to-date in 00:00
  remaining blocker: frontend state reuse still spends about 1m40s before
  backend partial emit; next work should make the frontend check-skip guard and
  hydrate preflight accept the current partial-merge proof without full replay

POST-QTLC1-INCREMENTAL-CHANGED-ONLY-NATIVE-EMIT-003
  fixed: checked-source execution can now hydrate the full typed set from the
  frontend-state product even when qtlc0 compiler identity changed but qtlc1
  source is stable. The build reports full-typed-set applied=true with
  parse=0, check=0, hydrate=422, then builder skipped.
  fixed: native unit-cache changed_paths now carries the full changed path set
  instead of an 8-path display sample, so changed-only emit receives every
  changed source unit.
  fixed: when checked-source/lowered restore is source-stable but qtlc0
  compiler identity changed, the driver derives changed paths from the
  lowered-restore complement instead of falling back to full emit.
  fixed: per-unit object cache records object paths by current lowered-unit
  index for all 422 units, not by body-only positional index. This prevents
  provider objects from being mapped to the wrong source unit.
  fixed: stale per-unit object manifests are no longer accepted for partial
  link. changed-only emit requires a current unit/object manifest; otherwise
  the build performs a safe refresh instead of linking wrong objects.
  fixed: native per-unit object action identity is bumped to
  native-object-unit-path-v2, so previously poisoned global object entries from
  the old positional manifest cannot be restored.
  proof: after qtlc0 rebuild, qtlc0 build -j8 qtlang/compiler reported
  checked-source full-typed-set applied=true files=422 parse=0 check=0
  hydrate=422, backend partial lowering restored_observed_units=413 with
  lower_plan_units=9, then performed one safe full object refresh because the
  old object cache identity was poisoned. It compiled_objects=422 once,
  wrote native-backend.per-unit-object-cache valid=true units=422
  split_ready_units=422, and built qtlang/compiler/build/debug/qtlc1.
  proof: immediate qtlc0 build -j8 qtlang/compiler returned status=up-to-date
  in 00:00, and qtlang/compiler/build/debug/qtlc1 version prints
  qtlc1 compiler 0.1.0.
  fixed: native per-unit object action identity is now
  native-object-unit-source-v3. The action key is unit source/body/target/runtime
  identity and no longer includes the package-wide native dependency key, so one
  source delta does not invalidate every object action.
  fixed: partial native emit validates either the current full manifest or the
  unchanged-unit subset of the previous manifest. Changed units compile fresh;
  unchanged unit objects are appended from the manifest/global cache.
  fixed: partial manifest writing derives stable per-unit object paths for all
  lowered units even when the native backend emitted assembly only for changed
  units. A previously package-bound manifest can recover by deriving
  <unit-index>.<module>.native.o paths from the manifest row order.
  fixed: backend handoff dry-lowering now validates hydrated AST roots/children
  before HIR/MIR lowering, so an invalid hydrated AST rejects the cache proof
  instead of throwing.
  proof: after qtlc0 rebuild, qtlc0 build -j8 qtlang/compiler reported
  lower_plan_units=9, restored_observed_units=413, changed_inputs=9,
  appended_from_manifest=413, compiled_objects=9, linked=1, and wrote
  native-backend.per-unit-object-cache split_supported=true
  split_ready_units=422 global_stored_units=422.
  proof: after the qtlc0 lowered-restore include split, qtlc0 build -j8
  qtlang/compiler rejected compiler-identity caches as expected, then reported
  native lowered-unit cache unchanged=413 changed=9, lower_plan_units=9,
  restored_observed_units=413, changed_inputs=9, appended_from_manifest=413,
  fresh_objects=9, compiled_objects=0, linked=1, and resealed
  native-backend.per-unit-object-cache split_ready_units=422
  global_stored_units=422.
  proof: a qtlc1 source-only edit in buildSystem/artifactPolicy.qn reported
  module-delta changed=1 affected=21, backend unchanged=412 changed=10,
  lower_plan_units=10, changed_inputs=10, and compiled_objects=10. The first
  attempt exposed a bad package-bound manifest; after recovery, qtlc0 build
  completed successfully without manual cache deletion.
  proof: immediate no-change qtlc0 build -j8 qtlang/compiler returned
  status=up-to-date in 00:00.
  remaining work: checked-source partial typed-base still reports
  applied=false with reason
  checked-source-partial-typed-base-failed:partial-typed-base-materialized, so
  one-file edits still run full source analysis before the backend delta path.
  Next work should make the partial frontend materializer/preflight accept the
  401/21 or 419/3 hydrate/recheck plan and skip parse/check for unchanged
  modules.

POST-QTLC0-BIG-FILE-SPLIT-001
  fixed: backend_dump_commands.cpp no longer owns all backend artifact/cache
  helpers inline. The first split moved artifact/package/install helpers,
  build-plan helpers, and resume/cache command helpers into focused include
  fragments.
  files:
    qtlc/compiler/driver/src/dumps/backend/backend_artifact_helpers.inc
    qtlc/compiler/driver/src/dumps/backend/backend_build_plan_helpers.inc
    qtlc/compiler/driver/src/dumps/backend/backend_cache_command_helpers.inc
    qtlc/compiler/driver/src/dumps/backend/backend_native_lowered_restore_parse_helpers.inc
    qtlc/compiler/driver/src/dumps/backend/backend_native_lowered_restore_probe.inc
    qtlc/compiler/driver/src/dumps/backend/backend_checked_source_cache_common.inc
    qtlc/compiler/driver/src/dumps/backend/backend_checked_source_cache_replay_probe.inc
    qtlc/compiler/driver/src/dumps/backend/backend_checked_source_cache_module_delta.inc
    qtlc/compiler/driver/src/dumps/backend/backend_checked_source_cache_tokens.inc
    qtlc/compiler/driver/src/dumps/backend/backend_checked_source_cache_ast.inc
    qtlc/compiler/driver/src/dumps/backend/backend_checked_source_cache_resolve.inc
    qtlc/compiler/driver/src/dumps/backend/backend_checked_source_cache_types.inc
    qtlc/compiler/driver/src/dumps/backend/backend_checked_source_cache_semantic.inc
    qtlc/compiler/driver/src/dumps/backend/backend_checked_source_cache_module_graph_records.inc
    qtlc/compiler/driver/src/dumps/backend/backend_checked_source_cache_hydration.inc
    qtlc/compiler/driver/src/dumps/backend/backend_checked_source_cache_materializer_dry_lower.inc
    qtlc/compiler/driver/src/dumps/backend/backend_checked_source_cache_frontend_gates.inc
    qtlc/compiler/driver/src/dumps/backend/backend_checked_source_cache_backend_handoff.inc
  proof: cmake --build qtlc/build --target qtlc -j8 passed after the split.
  proof: qtlc0 build -j8 qtlang/compiler passed after the lowered-restore split
  and immediate no-change build returned status=up-to-date in 00:00.
  proof: cmake --build qtlc/build --target qtlc -j8 passed after the
  checked-source cache split. qtlc0 build -j8 qtlang/compiler then reported
  native unit cache unchanged=413 changed=9, lower_plan_units=9,
  restored_observed_units=413, changed_inputs=9, appended_from_manifest=413,
  fresh_objects=9, compiled_objects=0, and immediate no-change build returned
  status=up-to-date in 00:00.
  current line counts: backend_native_lowered_restore_probe.inc and all
  backend_checked_source_cache*.inc fragments are now below the 2000-line
  target; backend_dump_commands.cpp is still about 6.7k lines. The next split
  should carve dump_backend_ir into function-phase fragments.
  fixed: early build-hydrate now trusts a valid live materializer artifact for
  full typed-set hydration, including compiler-identity rebuilds where the
  previous materializer compare key is stale. The full typed-set materializer
  derives the hydration key from the token artifact and requires all attached
  frontend/type/semantic/module-graph artifacts to match that key.
  proof: after qtlc0 rebuild, qtlc0 build -j8 qtlang/compiler reported
  materializer_available=true, checked-source full-typed-set applied=true
  files=422 parse=0 check=0 hydrate=422, checked-source builder skipped,
  post-hydrate probes skipped, native unit cache unchanged=413 changed=9,
  lower_plan_units=9, restored_observed_units=413, changed_inputs=9,
  appended_from_manifest=413, fresh_objects=9, compiled_objects=0, and build
  completed in 02:38. Immediate no-change build returned status=up-to-date in
  00:00.
  remaining speed blocker: backend lowering still spends time on the 9 changed
  qtlc0-affected units and emit still walks production-contract metadata. Next
  work should reduce changed-unit lowering cost and split backend_dump_commands
  into real translation units.

POST-QTLC1-INCREMENTAL-LARGE-CACHE-PROBE-SKIP-003
  fixed: qtlc0 no longer reads the stale multi-GB monolithic
  native-backend.product-cache during qtlc1 builds; probes now return
  native-backend-product-cache-skipped-large-file when the cache is above the
  monolithic byte limit, leaving shard/object caches as the source of truth
  fixed: native partial emit now applies for immediate partial shard restore as
  well as deferred restore; changed_paths drives changed_inputs=1 even when
  restored_units are already present before backend emit
  fixed: hidden native executable materialization no longer deduplicates static
  .a archive inputs; duplicate static archives are valid and required for
  one-pass Unix link resolution when recovered cached objects reference runtime
  symbols after the first archive occurrence
  proof target: restoring qtlang/compiler/main.qn reports
  native-backend-product-cache-skipped-large-file, changed_inputs=1,
  metadata_inputs=399, appended_from_manifest=287, compile_invocations=1,
  action_hit=true delivered=true link_skipped=true, emitted=2,
  compiled_objects=0, linked=0; no-change build then reports up-to-date in 00:00
  remaining blocker: frontend-state validation/hydration probes still dominate
  the one-file delta path before backend work starts

POST-QTLC1-INCREMENTAL-FRONTEND-PROBE-SKIP-004
  fixed: ordinary partial typed-base qtlc1 builds skip expensive post-check
  frontend execution probes after the real recheck is complete; full
  frontend-state/token/ast/resolve/type/semantic/module-graph hydration probes
  are still preserved when checked-source execution apply is requested or a
  full hydrated builder path is used
  proof target: one-file qtlc1 main.qn edit reports
  checked-source post-check frontend probes skipped hydrate=398 recheck=1,
  native-backend-product-cache-skipped-large-file, changed_inputs=1,
  metadata_inputs=399, emitted=2, and the build completes in about 27 seconds;
  restoring main.qn then reports action_hit=true delivered=true link_skipped=true
  and completes in about 26 seconds; no-change build remains up-to-date in 00:00
  remaining blocker: source analysis still parses/resolves 399 files before
  applying the partial typed base; next work should avoid full parse/resolve for
  unchanged modules by using frontend token/AST state before CheckedSourceBuilder

POST-QTLC1-INCREMENTAL-PREPARSE-HYDRATED-BASE-005
  fixed: CheckedSourceBuilder now attaches unchanged hydrated typed files before
  parse workers run when the partial typed-base plan is valid; unchanged modules
  are matched by normalized source path plus content hash, then skipped by
  lex/parse/resolve and reused during precheck metadata
  fixed: the later typed-file hydration step recognizes already attached
  preparse files and still schedules fallback recheck on any hydrate miss
  proof target: one-file qtlc1 main.qn edit reports
  reuse-preparse-hydrated-files unit=398/398, reuse-hydrated-typed-files
  unit=398/398, checked-source post-check frontend probes skipped, backend
  units=1, changed_inputs=1, compiled_objects=1; restoring the file reports
  lowered units=0, compiled_objects=0, linked=0, link_skipped=1; no-change build
  remains up-to-date in 00:00
  remaining blocker: qtlc0 still pays seconds in import closure/progress,
  precheck-resolve over the full module graph, and artifact-materialize even
  when link is skipped; next work should cache the resolved module graph and
  short-circuit artifact materialization on executable action hit

POST-QTLC1-INCREMENTAL-BINARY-LINK-HIT-MATERIALIZE-SKIP-006
  fixed: normal app binary builds no longer force package/interface metadata
  materialization only because a manifest has a library importRoot; binary app
  output remains the executable, while library/package metadata stays reserved
  for library targets, install, or non-binary artifact modes
  fixed: when the native executable action cache delivers the final binary,
  qtlc0 skips ArtifactEmitter source/object/link materialization and records the
  existing executable as the only required artifact for that normal binary path
  proof target: restoring qtlang/compiler/main.qn reports
  action_hit=true delivered=true link_skipped=true and
  native artifact materialization skipped reason=delivered-binary-link-cache;
  no-change build remains up-to-date in 00:00
  remaining blocker: qtlc0 still spends time before this point in source
  import closure, full precheck-resolve, backend cache probing, and summary/cache
  writes; next work should cache/resume the resolved module graph and make the
  source-dependency scanner avoid full module walks for one private body edit

POST-QTLC1-INCREMENTAL-PRECHECK-GRAPH-REUSE-007
  fixed: partial typed-base builds now reuse the hydrated semantic module graph
  when every rechecked module has the same import/export surface as the cached
  module; private body edits skip full resolve_semantic_program while public API
  or import changes fall back to the full graph resolver
  fixed: reused graph replacement preserves cached import resolutions on the
  rechecked module while replacing its current metadata/local symbol surface
  proof target: one-file qtlc1 main.qn private body edit reports
  precheck-resolve-reuse unit=399/399 tokens=1, reuse-plan valid=true, and
  source analysis completes in about 8 seconds; restoring main.qn also reports
  precheck-resolve-reuse plus action_hit=true delivered=true link_skipped=true
  and native artifact materialization skipped
  remaining blocker: import closure/source scan and backend cache probes still
  walk the whole package; next work should add a source-dependency scanner cache
  keyed by manifest/source-root list plus per-file hash so private body deltas do
  not rescan unchanged module boundaries

POST-QTLC1-INCREMENTAL-IMPORT-CLOSURE-REUSE-008
  fixed: CheckedSourceBuilder skips the import-closure lex/parse pass for
  package builds when a partial typed-base reuse plan is active and the driver
  already supplied a source path list covering every loaded file under the
  package root
  fixed: sparse/manual source builds still run import closure, and public
  import-surface changes are still guarded later by the semantic graph surface
  comparison before graph reuse is accepted
  proof target: one-file qtlc1 private body edit reports a single
  import-closure-reuse unit=399/399 event instead of the 50/100/.../399
  import-closure stream, then reports reuse-preparse-hydrated-files and
  precheck-resolve-reuse; restoring main.qn reports the same import-closure
  reuse plus link_skipped=true
  remaining blocker: source analysis still spends time in full precheck metadata
  progress over hydrated files and backend cache probes still inspect whole
  package-level cache indices; next work should avoid per-file precheck progress
  for hydrated files and make backend cache probes path-targeted for the changed
  unit set

POST-QTLC1-INCREMENTAL-PRECHECK-METADATA-BULK-REUSE-009
  fixed: hydrated files no longer emit per-file precheck metadata progress; the
  builder reuses their semantic module metadata directly and only collects
  untyped metadata for rechecked files
  proof target: one-file qtlc1 private body edit reports import-closure-reuse,
  reuse-preparse-hydrated-files, a single precheck-metadata start event, and
  precheck-resolve-reuse; the previous 50/100/.../399 precheck metadata progress
  stream is gone
  remaining blocker: backend unit/lowered-unit cache probes still compare all
  399 source hashes even when the checked-source delta already identified one
  changed path; next work should pass the checked-source changed path set into
  native cache probing and avoid rehashing hydrated unchanged units

POST-QTLC1-INCREMENTAL-BACKEND-PROBE-TARGETED-010
  fixed: native backend unit-cache probes now accept the checked-source recheck
  path set and avoid recomputing semantic hashes for hydrated unchanged units;
  unchanged paths are validated by cache membership while only the rechecked
  path is rehashed
  fixed: lowered-unit cache probing uses the same targeted path set for shard
  materializer readiness, avoiding unchanged-unit rehashing while preserving
  changed/missing detection for the rechecked unit
  proof target: one-file qtlc1 private body edit reports
  native-backend-unit-cache-partial-targeted with unchanged=398 changed=1
  missing=0; restoring main.qn reports the same targeted unit probe, lowered
  units=0, action_hit=true, delivered=true, link_skipped=true, and artifact
  materialization skipped
  remaining blocker: the driver still probes the stale large monolithic native
  product cache before using shard/unit caches; next work should bypass that
  probe entirely when partial typed-base reuse and targeted unit cache probing
  are already active

POST-QTLC1-INCREMENTAL-MONOLITHIC-PRODUCT-PROBE-BYPASS-011
  fixed: qtlc1 partial typed-base builds bypass the monolithic native product
  cache probe when targeted unit/shard probing is active; the old multi-GB
  product cache is not parsed or restored for one-file deltas
  proof target: one-file qtlc1 private body edit and restore both report
  targeted_skip=true and
  reason=native-backend-product-cache-skipped-targeted-partial before the
  targeted unit cache probe; restore still reports lowered units=0,
  action_hit=true, delivered=true, link_skipped=true, and artifact
  materialization skipped
  remaining blocker: source analysis still starts after preflight cache probes
  and the backend still emits package-level restored metadata for 399 lowered
  units; next work should introduce a qtlc1 delta fast-path summary that avoids
  package-wide progress/report work when changed_inputs is exactly one and link
  action cache delivers

POST-QTLC1-INCREMENTAL-BINARY-DELTA-COMPLETION-012
  fixed: delivered binary link-cache restores with zero object work now finish
  through a native delta completion fast path after build stamp/source metadata
  update; the driver skips the old global cache doctor/report-detail tail for
  this case
  proof target: restore qtlang/compiler/main.qn after a one-line private body
  edit reports lower_plan_units=0, restored_units=399, action_hit=true,
  delivered=true, link_skipped=true, artifact materialization skipped, and
  native delta completion fast path with skipped=global-cache-doctor,report-detail;
  the previous [REPORT] rendering build report phase is gone
  remaining blocker: the build still spends about seven to eight seconds before
  backend handoff doing source-key checks, partial typed-base setup, and source
  analysis entry work; next work should cache the source dependency key/hydrate
  index so qtlc1 one-file restores reach backend handoff without a package-wide
  frontend preflight cost

POST-QTLC1-BOOTSTRAP-001
  implemented: qtlc1 executable now has a real bootstrap driver command surface
  with help/version/check/build command facts, CLI bootstrap state, session
  state, and exit-code model
  proof: qtlc0 check qtlang/compiler/driver, qtlc0 check qtlang/compiler,
  qtlc0 build -j8 qtlang/compiler, and build/debug/qtlc1 exits 0 with
  newline-safe bootstrap output
  fixed in qtlc0: qtlc1 can keep strong Quantum style instead of weakening
  source. `CompilerCommand.help()` and `CompilerExitCode.ok()` remain
  receiverless associated calls, while `session.readyExit().code` uses a real
  implicit `this` receiver ABI. Cross-module public field/method surface is
  imported through explicit module exports/imports.
  fixed in qtlc0: partial native link replacement dedupes stale per-unit object
  paths by native unit identity, so one-file qtlc1 rebuilds do not link both old
  and new `main.native.o` objects.

POST-QTLC1-BOOTSTRAP-002
  started: qtlc1 has a compiler-local host argument surface in
  driver/hostArgs.qn. qtlc1 must not import std::os and must not add
  runtimePackage, stdlibPackage, or pluginPackages to quantum.toml; host/OS
  command input stays inside the qtlc1 compiler seed until qtlc1 can build
  qtlc2.
  proof: qtlc0 build -j8 qtlang/compiler scans only 401 qtlc1 compiler files,
  builds qtlang/compiler/build/debug/qtlc1, and the binary exits 0 through the
  qtlc1-local default check command.
  implemented: qtlc1 check now routes through driver/checkReport.qn and calls
  qtlc1 source, lexer, and parser proof APIs instead of printing only hardcoded
  status lines.
  proof: qtlc0 build -j8 qtlang/compiler builds 402 qtlc1 files, then
  qtlang/compiler/build/debug/qtlc1 exits 0 and reports source model, lexer
  subset, parser subset, and module boundaries checked.
  fixed in qtlc0: native runtime-support progress now reports through the scan,
  the scan has bounded traversal guards, and large native packages skip fixture
  result simulation so qtlc1 builds do not hang in a stale runtime-support
  heartbeat.
  fixed in qtlc0: plain Quantum-style interpolation now works for expression
  placeholders without the old `f` prefix. A normal string like
  `"target: {report.target}"` lexes/parses as interpolation only when the brace
  body contains an expression; old formatting placeholders like `"{}"` remain
  normal formatting strings. Native lowering now routes interpolation-produced
  `format(...)` calls through the std.io formatting path even when the result is
  used as a string value.
  proof: qtlc0 builds and runs
  qtlc/tests/fixtures/bootstrap_plain_interpolation_field_access_ok.qn, printing
  `target: qtlang/compiler`; qtlc1 main output now uses
  `println("target: {report.target}")` style and qtlc0 build -j8
  qtlang/compiler produces a qtlc1 binary reporting 403 source files.
  started: qtlc1 check report now uses qtlc1-owned manifest/source scan model
  instead of keeping source count in the driver. `moduleSystem/manifest.qn`
  describes the qtlc1 manifest, `source/packageScan.qn` carries source-root,
  qn-file, and module-boundary scan facts, and driver/checkReport consumes
  those facts before lexer/parser proof checks.
  proof: qtlc0 check -j8 qtlang/compiler passes with 403 files; after removing
  the stale global executable cache entry and rebuilding, qtlc1 exits 0 and
  reports source files: 403 with source model, lexer subset, parser subset, and
  module boundaries checked.
  discovered qtlc0 backend cache debt: adding a new qtlc1 source file can leave
  partial link inputs missing the new object, and stale executable qcx entries
  can still be selected by action key even after temp-cache cleanup; qtlc0 must
  include new-unit object membership and stronger body identity in the link key.
  fixed in qtlc0: global native executable link-cache action key now includes
  source dependency, native dependency, runtime dependency, and source-map count
  in the link-policy identity instead of using only output path/target/profile.
  proof: after rebuilding qtlc0 and deleting only qtlc1 output artifacts, qtlc0
  reports native link executable cache action_hit=false with a new action hash
  instead of delivering the old stale qcx; rebuilt qtlc1 exits 0 and reports
  source files: 403.
  implemented: qtlc1 source/package scan no longer embeds qtlc1's source count,
  module-boundary count, manifest path, or source root in driver code. It calls
  compiler-host facts (`qtlc1HostSourceFileCount`,
  `qtlc1HostModuleBoundaryCount`, manifest/source-root/app-main probes), and
  qtlc0 folds those calls into literals from the semantic program being built.
  proof: qtlc0 rebuilds, qtlc0 build -j8 qtlang/compiler reports a partial
  delta for qtlang/compiler/source/packageScan.qn only, and the resulting
  build/debug/qtlc1 exits 0 with source files: 403 and module boundaries:
  checked.
  fixed in qtlc0: missing-output qtlc1 rebuild no longer recompiles native
  objects when the executable link cache is delivered. The delivered-binary
  path preserves the existing per-unit object cache instead of rewriting it.
  proof: after deleting only build/debug/qtlc1 and its buildstamp, qtlc0 build
  -j8 qtlang/compiler reports action_hit=true, delivered=true,
  link_skipped=true, compiled_objects=0, and
  reason=delivered-binary-link-cache-object-cache-preserved; rebuilt qtlc1
  exits 0.
  fixed in qtlc0: native binary link-cache metadata now records the delivered
  executable action/content hash, and precheck can restore a missing binary
  directly from that metadata before source analysis. The link executable cache
  helpers were split out of the large backend dump file into
  backend_native_link_executable_cache.inc, and per-unit native object cache
  helpers were split into backend_native_per_unit_object_cache.inc.
  proof: after deleting only build/debug/qtlc1 and its buildstamp, qtlc0 build
  -j8 qtlang/compiler reports native link executable early cache
  action_hit=true, delivered=true, link_skipped=true, elapsed=00:00, and
  skipped=source-analysis,lowering,emit,link; rebuilt qtlc1 exits 0.
  fixed in qtlc0: source-key changes that do not change backend semantic hashes
  now exit in precheck before checked-source analysis. qtlc0 probes the native
  unit cache against loaded source files, updates buildstamp/source metadata,
  and skips checked-source/lowering/emit/link when every native semantic hash is
  unchanged.
  proof: adding then removing a whitespace-only line in
  qtlang/compiler/driver/exitCode.qn reports native semantic precheck cache
  valid=true, cached=403, unchanged=403, changed=0, missing=0, elapsed=00:00,
  and skipped=checked-source,lowering,emit,link; qtlc1 exits 0.
  discovered qtlc0 cache debt: stale executable link-cache entries can deliver
  old qtlc1 binaries if the action key is incomplete; proof runs must reject or
  invalidate stale delivered binaries until the link action key includes enough
  source/body identity.
  implemented: production qtlc1 source tree no longer owns proof sources.
  Proof files live under qtlang/compiler/tests, qtlang/compiler/quantum.toml
  excludes tests/**, and qtlang/compiler/tests has a separate qtlc1-tests
  manifest plus testRunner/main.qn that calls executable proof functions.
  proof: qtlc0 check -j8 qtlang/compiler/tests passes with 412 files,
  qtlc0 build -j8 qtlang/compiler builds 359 production files, and
  build/debug/qtlc1 check qtlang/compiler reports status: ok.
  fixed in qtlc0: native executable proof runner now builds and runs from
  qtlang/compiler/tests. Imported enum variants used through field syntax, for
  example NormalControlFlowKind.Return, resolve from semantic enum metadata and
  lower to the correct native tag in comparisons and calls. Native package enum
  layouts are attached package-wide so enum definitions from imported modules
  are available to body lowering.
  fixed in qtlc0 cache: native lowered-unit shards, per-unit object restore,
  and link executable restore are guarded by backend_cache_version. Native
  object materialization no longer trusts local .native.o mtime freshness when
  an action key is present; it must restore by action key or compile from the
  current .native.s. This prevents stale objects from producing a stale linked
  executable after compiler/backend changes.
  fixed in qtlc0 native entry selection: when a package contains production
  compiler main and tests/main.qn, the native process entry now selects the
  tests/main.qn proof runner and matches the backend symbol by source range.
  proof: qtlc0 build -j8 qtlang/compiler/tests succeeds, reports v8 native
  cache miss/rebuild with compiled_objects=303 on first corrected build, and
  qtlang/compiler/tests/build/debug/qtlc1-tests exits 0. qtlc0 build -j8
  qtlang/compiler succeeds/up-to-date, and ./qtlang/compiler/build/debug/qtlc1
  check qtlang/compiler exits 0 with lexer/parser/module boundaries checked.

POST-QTLC0-NATIVE-CROSS-MODULE-DIRECT-CALL-001
  fix qtlc0 native direct body target table so imported public functions from
  explicit and wildcard module exports are available during native lowering.
  Required proof: qtlc0 build -j8 qtlang/compiler/tests succeeds and
  qtlang/compiler/tests/build/debug/qtlc1-tests exits 0.
  status: complete.

POST-QTLC0-NATIVE-ENUM-LAYOUT-PROOF-RUNNER-001
  fix qtlc0 native imported enum layout/variant lowering for executable proof
  code and prove field-style enum variants lower to the correct native tag.
  status: complete.

POST-QTLC0-NATIVE-ACTION-CHECKED-OBJECT-CACHE-001
  require action-checked restore/compile for native per-unit objects when a
  global action key exists; local mtime is not proof after backend changes.
  status: complete.

POST-QTLC1-BOOTSTRAP-003
  qtlc1 builds qtlc2 artifact

POST-QTLC1-FRONTEND-DATA-CONTRACT-001
  move frontend declarations from count-only products toward arena/index state.
  status: started.
  implemented: DeclarationSet carries Slice<DeclarationId> rows,
  DeclarationArena stores Arena<DeclarationRecord> and ID slices,
  DeclarationIndex stores DeclarationKey/DeclarationId rows, and
  DeclarationTable owns arena/index state.
  proof: qtlc0 check -j8 qtlang/compiler passes with 425 files.
  remaining: real parser-produced declaration records, Option/Result lookup
  APIs, and final Index<DeclarationKey, DeclarationId> once qtlc0 generic field
  substitution supports mixed generic key/value range fields cleanly.

POST-QTLC1-BOOTSTRAP-004
  qtlc2 checks qtlc1 source; deterministic qtlc1-owned products remain pending
```

## Final qtlc1 Gate

```text
POST-QTLC1-FINAL-001
  run full qtlc0 -> qtlc1 -> qtlc2 self-host proof, including qtlc2 check

POST-QTLC1-FINAL-002
  run magic-fast no-change, one-private-body-change, one-public-API-change,
  and dependency-source-avoidance benchmark proof

POST-QTLC1-FINAL-003
  freeze qtlc1 scope and move runtime/stdlib/toolchain assembly to qtlc2
```


 
### POST-QTLC0-POINTER-PROVENANCE-AUTHORITY-006

Centralize pointer authority and provenance before qtlc1 relies on pointer
facts: exclusive mutable authority, CFG validity narrowing, const enforcement,
fact-backed validity checks, and compiler-core-only legacy dereference.

### POST-QTLC0-POINTER-AUTHORITY-SSA-007

Close pointer alias and escape holes before qtlc1 self-hosting:

- mutable `*T` is a non-copyable exclusive authority token
- `^pointer` transfers authority explicitly
- `*const T` aliases preserve owner provenance
- local-derived pointers require a typed `noescape` call contract
- validity is `Unknown`, `Proven`, or `Invalid`
- pointer reassignment invalidates the active CFG proof
- native lowering cannot infer full validity from initializer shape

### POST-QTLC0-POINTER-AUTHORITY-EXPRESSION-GRAPH-008

Track pointer authority through the full MIR expression graph:

- reject local-derived pointer escape through calls, explicit moves, structs,
  tuples, and arrays
- reject aggregate returns that contain local-derived pointers
- mark the source pointer authority invalid after `^pointer`
- preserve the transferred destination provenance without runtime dispatch
- include ownership and pointer-lowering semantics in backend cache identity

### POST-QTLC0-POINTER-DEEP-OOP-SURFACE-PROOF-007

Implemented: deep generic object pointers with nested `impl view`, generic
roles, bounded blanket extensions, `context`, `extract`, and cross-module
extension dispatch. Mutable/const authority builds and runs through direct
native field loads/stores and SROA view inlining.

The implementation also fixed qtlc0 generic aggregate construction, raw-pointer
method receiver lowering, and partial-link object selection by stable source
identity instead of unstable unit ordinal.

### POST-QTLC0-POINTER-FULL-TYPE-MATRIX-PROOF-008

Prove mutable and const pointers with literal-first source across:

- primitive, wide integer, and floating payloads;
- `Option`, `Result`, `Vec`, `Array`, `Map`, range/slice/index carriers;
- decimal, complex, vector, matrix, tensor, bit, bits, qubit, and qreg fields;
- generic views and nested custom objects;
- direct native field loads/stores without reflection, allocation, or dynamic
  dispatch.

Any exposed gap is fixed centrally in qtlc0; qtlc1 source is not weakened.

Status: complete.

Implemented:

- scalar pointer loads/stores select width and ABI through the central native
  type classifier, including narrow, pointer-width, wide integer, and float
  families;
- aggregate pointer storage strips reference, mutable, const, and raw pointer
  qualifiers centrally before layout lookup;
- nested pointer-backed generic views preserve the original aggregate address
  instead of reading temporary stack copies;
- collection, decimal, complex, math-shape, bit, and quantum proof products
  carry their runtime support through central type-family scanning;
- literal-first `Option`, `Result`, `Vec`, `Array`, `Map`, nested object,
  vector, matrix, tensor, bit, bits, qubit, and qreg construction builds
  without weakening qtlc1 source.

Proof:

- `qtlc build -j8 qtlang/lowlavel/qtlc0` succeeds;
- `qtlc0_explicit_ownership_symbols` prints every pointer proof as `ok` and
  exits `0`;
- generated native assembly contains direct field loads/stores with no
  indirect dispatch, reflection lookup, named-argument map, or allocation in
  the pointer hot path;
- the immediate no-change rebuild reports `outputs up to date` in `00:00`.

### POST-QTLC0-LARGE-AGGREGATE-READ-ALIAS-009

Lower aggregate locals initialized from an existing local or field as
pointer-sized read-borrow aliases:

- preserve qtlc1's rich arena/index/declaration data model
- avoid repeated multi-megabyte stack copies
- keep fresh literals and constructor results in owned storage
- retain direct native field loads with no heap allocation or dispatch

### POST-QTLC0-OWNERSHIP-LITERAL-FULL-MATRIX-PROOF-009

Status: complete.

Implemented:

- executable default read-borrow through `value`;
- executable explicit shared borrow through `&value`;
- executable mutable borrow through `&!value`;
- executable ownership transfer through `^value`;
- generic nested custom aggregate construction, reading, mutation, and move;
- real negative fixtures for use-after-move, move-while-borrowed,
  shared/mutable alias conflicts, and mutation during a shared borrow;
- generic view-chain type substitution in qtlc0 type checking;
- central effective field storage width after generic type substitution.

Proof:

- `qtlc check qtlang/lowlavel/qtlc0` succeeds;
- native build and executable run succeed with ownership status `0`;
- four registered ownership diagnostic tests pass;
- pointer/ownership assembly contains no indirect dispatch, reflection lookup,
  named-argument map, or hot-path allocation;
- no-change native build completes as up to date in `00:00`.

### POST-QTLC0-GENERIC-AGGREGATE-CONCRETE-LAYOUT-010

Status: complete.

Implemented:

- concrete native layouts for generic aggregates whose payloads are wider than
  one machine word;
- recursive generic substitution for nested aggregate fields;
- exact monomorph call-target selection using concrete parameter signatures;
- correct two-register text/bytes returns and aggregate sret returns;
- concrete nested aggregate receiver addresses for generic `impl view` chains;
- size-gated forwarding from concrete generic aggregates to compatible erased
  pointer ABIs;
- direct native lowering only, with no C, C++, LLVM, reflection, or dynamic
  dispatch fallback.

Proof:

- `LayoutCell<string>` places `sentinel` after the two-word string payload;
- `LayoutCell<LayoutPair>` preserves both aggregate words and the sentinel;
- `LayoutEnvelope<string>` resolves nested `cell.current` and `cell.intact`;
- the complete low-level ownership, pointer, generic OOP, role, context,
  extract, and extension executable exits `0`;
- generated hot-path assembly uses concrete offsets, direct loads/stores, and
  concrete monomorphs without allocation.

### POST-QTLC0-NATIVE-OBJECT-COMPONENT-IDENTITY-011

Status: complete.

Contract:

- per-unit object actions use emitted assembly content, target, profile,
  object format, and runtime ABI inputs;
- full qtlc0 executable identity is not part of an object action;
- a compatible older manifest may supply an object only when its recorded
  native assembly hash exactly matches the current unit assembly;
- changed assembly receives a new action and recompiles normally;
- unchanged units survive qtlc0 implementation rebuilds without recompiling;
- native backend only, with no generated C, C++, or LLVM path.

Proof target:

- rebuild qtlc0 after a backend-only implementation change;
- build `qtlang/compiler`;
- report restored objects for unchanged units rather than
  `compiled_objects=436`;
- immediate no-change build completes in `00:00`;

### POST-QTLC0-NATIVE-OBJECT-CODEGEN-IDENTITY-001

Status: complete.

Contract:

- every native per-unit object action includes the componentized native
  code-generation identity;
- native backend or ownership implementation changes cannot restore objects
  produced by older code-generation logic;
- object manifests must match the current code-generation identity before
  they can restore objects, rebuild link inputs, or restore an emit/link plan;
- frontend-only qtlc0 changes do not invalidate native objects;
- unchanged source with unchanged native codegen still restores per-unit
  objects normally.

Proof:

- stale source-snapshot, semantic-unit, build-stamp, and checked-source early
  exits now require a current native object manifest;
- changing native emission rejected the previous executable and compiled two
  fresh proof objects instead of restoring objects from the older emitter;
- wide byte reads and writes preserve their aggregate receiver across runtime
  bridge calls;
- 64-bit big-endian byte frames emit `bswapq`, not a truncating `bswapl`;
- `qtlc0_native_bytes_hot_path` builds and exits successfully;
- the immediate unchanged rebuild completes in `00:00`.
- resulting qtlc1 checks `qtlang/compiler` successfully.

Proof:

- after rebuilding qtlc0, qtlc1 binary/link metadata was removed while
  lowered-unit and global object caches were preserved;
- reconstruction reported `restored=436 compiled=0`;
- the incremental summary reported
  `restored_objects=436 compiled_objects=0`;
- qtlc1 linked successfully from the restored objects.

### POST-QTLC0-NATIVE-EMIT-PRODUCT-RESTORE-012

Status: complete.

Contract:

- when every native semantic unit is unchanged and the final executable is
  missing, restore the link product directly from cached unit objects;
- validate every manifest row against the current source semantic hash;
- prefer global object-cache paths and preserve deterministic unit order;
- restore runtime and dependency inputs from the persisted native link plan;
- resolve runtime `.qca` metadata to the sibling linkable `.a` archive;
- apply the normal native math and C++ runtime link inputs;
- write the executable and build stamp without entering lower or emit;
- native backend only, with no generated C, C++, or LLVM source path.

Proof:

- removed only qtlc1, its build stamp, and native link metadata;
- preserved 436 lowered shards, the per-unit object manifest, global objects,
  and the persisted link plan;
- reconstruction reported
  `native emit products restored units=436 link_inputs=437 compiled_objects=0`;
- no `[LOWER]` or `[EMIT]` phase ran;
- reconstruction completed in `00:43` instead of the previous multi-minute
  full emit walk;
- the immediate no-change build completed in `00:00`;
- the restored qtlc1 checked all 436 compiler source files successfully.

### POST-QTLC0-NATIVE-PRECHECK-EMIT-RESTORE-013

Status: complete.

Contract:

- attempt missing or stale native executable reconstruction before
  `CheckedSourceBuilder`;
- require the loaded source-only dependency key to match existing metadata;
- recompute every loaded file's backend semantic hash and require an exact
  match with its per-unit object manifest row;
- reject missing, duplicate, extra, stale, or invalid native object units;
- restore runtime and dependency inputs from the persisted native link plan;
- relink the validated global objects directly and write the output stamp;
- fall back to normal checked-source analysis on any validation or link miss;
- do not replay qtlc1 modules that depend on changed qtlc0 host intrinsics
  merely to reconstruct an unchanged native executable.

Proof:

- rebuilt qtlc0, then removed only qtlc1, its build stamp, and native link
  metadata;
- preserved frontend artifacts, lowered shards, global objects, object
  manifest, and persisted link plan;
- precheck reported
  `native emit products restored in precheck units=436 link_inputs=437
  compiled_objects=0 skipped=checked-source,lowering,emit`;
- no `[CHECK]`, `[PARSE]`, `[LOWER]`, or `[EMIT]` phase ran;
- reconstruction completed in `0.84` seconds instead of the previous
  `00:43`, removing roughly 38 seconds of frontend hydration/typecheck;
- immediate no-change build completed in `00:00`;
- restored qtlc1 checked all 436 compiler files successfully.

### POST-QTLC1-SELFHOST-PLATFORM-IO-001

Status: complete.

Contract:

- provide a compiler-owned native platform boundary without importing runtime
  or stdlib packages into qtlc1 source;
- discover package `.qn` paths recursively at qtlc1 execution time;
- sort source rows deterministically and expose path, module path, byte length,
  and content-fingerprint rows;
- read source text and query file existence through the compiler host ABI;
- create output directories recursively;
- support atomic text writes, rename, and process execution for later native
  object and linker stages;
- keep the ABI implementation separate from the large general platform ABI;
- replace qtlc0-folded `hostAllSource*` constants in `SourceFileTable` with
  qtlc1 runtime `CompilerIo` calls.

Proof:

- qtlc0 platform and source module checks passed;
- qtlc0 built the 439-file qtlc1 compiler with the native backend;
- qtlc1 checked all 439 compiler source files successfully;
- binary disassembly contains direct calls to
  `QnCompilerSourcePaths`, `QnCompilerSourceModulePaths`,
  `QnCompilerSourceByteLengths`, and `QnCompilerSourceFingerprints`;
- qtlc1 build atomically materialized
  `build/debug/qtlc2.build-plan` with its stage, target, source, and product
  state;
- binary disassembly contains direct calls to
  `QnCompilerCreateDirectories`, `QnCompilerAtomicWriteText`, and
  `QnCompilerRename`;
- the compiler-host symbols remain private to the compiler host boundary and
  are not added to the frozen public language runtime ABI;
- the immediate qtlc0 no-change rebuild completed in `00:00`.

Boundary:

- this completes the self-host platform I/O prerequisite;
- real tokenization, parsing, HIR/MIR lowering, native object emission, and
  qtlc2 linking remain separate implementation steps.

### POST-QTLC0-INCREMENTAL-PIPELINE-FREEZE-001

Status: complete and frozen.

Locked baseline:

- immediate no-change qtlc1 build completes in `00:00`;
- a one-source semantic delta hydrates 456 modules and rechecks 3;
- only 1 native unit is lowered and only 1 object is compiled;
- 458 lowered/object units are restored;
- ABI-only metadata closure restore reduced payload restore from about
  `8.58s` to about `0.145s`;
- the measured one-source delta build completes in about `19s`;
- native artifact work is no longer the primary bottleneck:
  object materialization is about `0.012s` and linking about `0.402s`.

Freeze rule:

- qtlc0 cache, hydration, lowering, emit, object, and link code is frozen;
- qtlc1 implementation must proceed without opportunistic qtlc0 performance
  changes;
- qtlc0 may be changed only for a reproducible correctness or language
  capability blocker required by qtlc1;
- every permitted qtlc0 change must preserve the no-change and one-source
  delta proof above.

Next:

- `POST-QTLC1-REAL-LEXER-PARSER-002`.

### POST-QTLC1-REAL-LEXER-PARSER-002

Status: complete.

Implemented:

- `RealFrontend` obtains the deterministic package source list through
  `CompilerIo`;
- every source file is read by qtlc1 at execution time;
- `SourceLexer` produces owned `SyntaxToken` and `Trivia` vectors with
  `FileId` and `SourceSpan`;
- `SourceParser` consumes each real token stream and produces a per-file
  `AstArena`;
- `PackageLexProduct` and `PackageAstProduct` own the complete package
  frontend products;
- `FrontendProduct.forPackage` uses these real products instead of host lexer
  and parser count intrinsics.

Boundary:

- lexer/parser products are real qtlc1-owned state;
- declaration construction still summarizes parser products into counts;
- the next item must create concrete parser-produced declaration records and
  make declaration counts derived views only.

Next:

- `POST-QTLC1-REAL-DECLARATION-ROWS-003`.

### POST-QTLC1-REAL-DECLARATION-ROWS-003

Status: complete.

Implemented:

- `SourceParser` emits concrete `AstDeclaration` rows from real token streams;
- `PackageAstProduct` owns the complete declaration vector;
- `DeclarationRecord.fromAst` converts parser declarations into frontend
  declaration records;
- `DeclarationRecordBatch` groups concrete records by declaration kind without
  integer-to-enum remapping;
- declaration arenas, tables, and indexes consume concrete parser-owned rows;
- qtlc0 now respects lexical local bindings when a local name shadows an
  implicit receiver field.

Verification:

- `qtlc build -j8 qtlang/qtlc0_test_build` passes and its executable runs;
- `qtlc build -j8 qtlang/compiler` passes;
- `qtlc1 check qtlang/compiler` passes across 460 source files.

Next:

- build the real module/name-resolution graph from concrete declaration rows.

### POST-QTLC1-CORE-BYTES-FOUNDATION-001

Status: implemented source contract.

Contract:

- add `compiler/core/bytes` as the compiler-owned binary foundation;
- lock `b"..."`, target-typed byte-list, and empty byte literals;
- separate owned bytes, immutable views, mutable views, buffers, and cursors;
- provide checked bool, integer through 256-bit, float, varint, alignment,
  slicing, seeking, append, and finish operations;
- allow default little-endian operation and explicit byte order;
- provide statically dispatched `ByteCodec<T>` and `toBytes` extensions;
- keep runtime/stdlib packages outside the qtlc1 core dependency boundary.

Verification:

- `qtlc check qtlang/compiler/core` passes with 26 files.

Next:

- `POST-QTLC0-NATIVE-BYTES-HOT-PATH-002` binds the surface to native
  intrinsics and adds executable/assembly proofs.

### POST-QTLC0-PER-FILE-FRONTEND-SHARDS-001

Status: complete.

Contract:

- store token, AST, and type products in stable per-file shards;
- use normalized source path as shard identity and source hash as validity;
- keep a compact shard index for lookup without opening every payload;
- preserve unchanged shards across compiler and application builds;
- update only changed source shards during partial frontend reuse;
- keep the existing monolithic artifacts as compatibility fallback.

Proof:

- initial sealing wrote 459 frontend shards;
- a one-file semantic change wrote 1 shard and reused 458;
- lowering processed 1 unit and restored 458 lowered units;
- native object materialization produced 1 fresh object and restored 458;
- reverse proof also wrote 1 shard and reused 458;
- immediate no-change build completed in `00:00`;
- proof source was restored to its original value.

Next:

- `POST-QTLC0-PARALLEL-FRONTEND-SHARD-HYDRATION-002` should hydrate unchanged
  token, AST, and type shards in parallel and remove the remaining monolithic
  15–17 second partial frontend decode.

### POST-QTLC0-PARALLEL-FRONTEND-SHARD-HYDRATION-002

Status: complete.

Contract:

- preserve the global source-file index in every frontend shard;
- validate shard identity by stable path, source hash, and file index;
- read token, AST, and type shards through a bounded worker pool;
- reconstruct the validated frontend products from per-file shards;
- attach AST and type products concurrently after source/token hydration;
- retain the monolithic frontend artifacts as a compatibility fallback.

Proof:

- the v2 shard set contains 459 globally indexed source units;
- a one-file semantic change loaded the partial typed base from parallel
  shards;
- frontend analysis dropped from the previous 15–18 seconds to 13 seconds;
- the change wrote 1 frontend shard and reused 458;
- lowering processed 1 unit and restored 458 lowered units;
- native object materialization produced 1 fresh object and restored 458;
- reverse proof produced the same result and restored the proof source;
- immediate no-change build completed through the output-current gate.

Next:

- `POST-QTLC0-PER-FILE-SEMANTIC-METADATA-SHARDS-003` should persist complete
  semantic metadata per module. The current partial hydration still repairs
  semantic metadata for 428 modules, which is now the dominant frontend cost.

### POST-QTLC0-PER-FILE-SEMANTIC-METADATA-SHARDS-003

Status: complete.

Contract:

- persist complete semantic metadata in stable per-module shards;
- include imports, functions, structs, enums, classes, interfaces, traits,
  impl layouts, nested members, attributes, derive plans, artifact plans,
  runtime metadata, type IDs, and source ranges;
- key shards by stable module identity and validate them with source hashes;
- hydrate metadata shards through a bounded worker pool;
- merge previously missing metadata categories while retaining trusted
  semantic-program impl dispatch state;
- keep AST metadata reconstruction as a compatibility fallback only.

Proof:

- 459 versioned semantic metadata shards were sealed;
- one-file forward and reverse changes each loaded 459 metadata shards;
- both runs reported `repaired=0`;
- each delta wrote 1 metadata shard and reused 458;
- module delta remained narrow at 3 affected modules or less;
- lowering processed 1 source unit and restored 458 lowered units;
- native object materialization produced 1 fresh object and restored 458;
- forward build completed in `00:36`;
- reverse build completed in `00:42`;
- immediate no-change build completed through the output-current gate;
- proof source was restored.

Next:

- `POST-QTLC0-DIRECT-PER-FILE-FRONTEND-DECODE-004` should deserialize token,
  AST, and type shards directly into per-file compiler products instead of
  reconstructing temporary monolithic artifact text. This is the remaining
  major cost inside the 13–15 second frontend hydration phase.

### POST-QTLC0-DIRECT-PER-FILE-FRONTEND-DECODE-004

Status: complete.

Contract:

- decode every token, AST, and type shard in an isolated per-file worker;
- preserve stable global `SourceFileId` values after local shard validation;
- restore token lexeme views against the final source buffer;
- restore AST source ranges to their global file IDs;
- materialize type contexts, node types, and backend facts independently;
- merge decoded files in stable package order;
- retain aggregate shard reconstruction and monolithic artifacts as validated
  compatibility fallbacks.

Proof:

- direct per-file frontend materialization loaded all 459 source units;
- semantic metadata hydration loaded 459 shards with `repaired=0`;
- frontend analysis dropped from 13–15 seconds to 8 seconds;
- forward and reverse one-file builds each wrote 1 frontend shard and reused
  458;
- each delta lowered 1 source unit and restored 458 lowered units;
- each delta produced 1 fresh native object and restored 458;
- forward and reverse builds completed in `00:32`;
- immediate no-change build completed through the output-current gate;
- proof source was restored.

Next:

- `POST-QTLC0-FRONTEND-SHARD-BINARY-CODEC-005` should replace line-oriented
  token, AST, and type shard records with compact binary records. Direct
  decoding is parallel now, but text parsing and hex decoding still dominate
  the remaining 8-second frontend phase.

### POST-QTLC0-FRONTEND-SHARD-BINARY-CODEC-005

Status: complete.

Contract:

- replace newline-delimited frontend shard files with a versioned binary v3
  container;
- encode shard identity, source hash, file record, record-group counts, and
  records with fixed-width little-endian lengths;
- decode token, AST, and type groups directly without scanning or classifying
  lines;
- reject truncated files, invalid group counts, stale source hashes, trailing
  bytes, and file identity mismatches;
- invalidate v2 shard indexes once and preserve one-file shard reuse after the
  migration.

Proof:

- qtlc0 rebuilt successfully with the binary codec;
- all 459 frontend shards were resealed once as v3 binary files;
- the reverse one-file delta loaded the direct per-file path;
- the delta wrote 1 frontend shard and reused 458;
- semantic metadata loaded 459 shards with `repaired=0`;
- lowering processed 1 unit and restored 458;
- native object materialization produced 1 fresh object and restored 458;
- the delta build completed in `00:31`;
- frontend analysis remained `00:08`;
- the immediate no-change build completed in `00:00`;
- proof source was restored.

Finding:

- binary framing removes line scanning and record classification, but the
  remaining frontend time is dominated by parsing the textual fields and hex
  payloads inside each binary record.

Next:

- `POST-QTLC0-FRONTEND-SHARD-STRUCTURED-BINARY-006` should encode token, AST,
  type, range, identifier, and fact fields as native binary values so direct
  hydration performs no metadata-field parsing or hex decoding.

### POST-QTLC0-FRONTEND-SHARD-STRUCTURED-BINARY-006

Status: complete.

Contract:

- encode token kind and source offsets as fixed-width native fields;
- encode AST kind, ranges, text, alias, operator, flags, and child IDs without
  textual records or hex payloads;
- encode type context rows, node-type bindings, fact counts, and supported
  backend type facts as typed binary values;
- build `Token`, `AstNode`, and `TypeCheckResult` objects directly in each
  per-file worker;
- remove temporary token, AST, and type artifact text construction from the
  direct hydration path;
- encode the 459-file shard index as binary records;
- retain monolithic artifacts as the compatibility fallback.

Proof:

- qtlc0 rebuilt successfully with the structured v4 codec;
- all 459 frontend shards were resealed once;
- the reverse one-file delta used direct per-file structured hydration;
- the delta wrote 1 frontend shard and reused 458;
- semantic metadata loaded 459 shards with `repaired=0`;
- lowering processed 1 unit and restored 458;
- native object materialization produced 1 fresh object and restored 458;
- measured frontend analysis varied between `00:07` and `00:09`;
- measured delta builds varied between `00:31` and `00:36`;
- the immediate no-change build completed in `00:00`;
- proof source was restored.

Finding:

- record parsing and hex decoding are removed from direct hydration, but they
  were not the dominant remaining cost;
- source-file reads, per-file allocations, semantic metadata attachment, and
  the three-module recheck now dominate the frontend phase.

Next:

- `POST-QTLC0-FRONTEND-SHARD-ZERO-COPY-PROFILE-007` should instrument source
  reads, binary decode, object construction, semantic metadata attachment, and
  recheck separately before selecting mmap, pooled allocation, or lazy
  hydration work.

### POST-QTLC0-FRONTEND-SHARD-ZERO-COPY-PROFILE-007

Status: complete.

Contract:

- measure binary index decoding separately;
- measure shard file reads and structured decoding as accumulated worker CPU;
- measure source reads and compiler-object construction separately;
- record bounded-worker wall time and stable merge time;
- measure semantic-program hydration, module-graph hydration, semantic-file
  attachment, and semantic-metadata shard hydration separately;
- measure partial hydration wall time and changed-module recheck time;
- emit one compact cache profile line without verbose per-file output.

Forward one-file profile:

- shard index decode: `928 us`;
- shard reads: `39,692 us` accumulated worker CPU;
- structured decode: `165,696 us` accumulated worker CPU;
- source reads: `7,015 us` accumulated worker CPU;
- object construction: `228,061 us` accumulated worker CPU;
- complete eight-worker shard wall time: `66,926 us`;
- stable merge: `2,133 us`;
- semantic-program hydration: `543,066 us`;
- module-graph hydration: `2,596,465 us`;
- semantic-file attachment: `10,537 us`;
- semantic-metadata shards: `23,224 us`;
- complete partial hydration: `3,280,097 us`;
- three-module recheck: `1,951,585 us`.

Reverse one-file profile:

- complete eight-worker shard wall time: `64,253 us`;
- semantic-program hydration: `536,651 us`;
- module-graph hydration: `2,598,860 us`;
- semantic-metadata shards: `23,418 us`;
- complete partial hydration: `3,273,429 us`;
- three-module recheck: `1,994,415 us`.

Finding:

- mmap and zero-copy frontend shards are not the next priority: all 459
  structured shards hydrate in about `64–67 ms` wall time;
- module-graph hydration is the dominant frontend cost at about `2.60 s`;
- the module graph still scans and parses textual node, edge, diagnostic, and
  16,756 visibility records;
- changed-module recheck is the second major cost at about `1.95–1.99 s`;
- semantic-program hydration is third at about `0.54 s`.

Next:

- `POST-QTLC0-MODULE-GRAPH-STRUCTURED-BINARY-008` should persist and directly
  materialize module nodes, edges, topological order, visibility rows, and
  diagnostics as typed binary records, eliminating the current textual graph
  parser and hex decoding.

### POST-QTLC0-MODULE-GRAPH-STRUCTURED-BINARY-008

Status: complete.

Contract:

- replace the line-oriented module-graph cache with a versioned binary v2
  container;
- encode graph identity, validity, dependency key, cycle state, and duplicate
  identity state as typed fields;
- encode topological order, nodes, edges, import/exclusion items, source-root
  paths, visibility facts, and diagnostics without textual parsing;
- reconstruct `SemanticModuleGraph` directly from binary records;
- validate the binary graph against the checked-source shape and dependency
  key;
- retain the v1 text reader as a compatibility fallback;
- reseal a v1 graph during partial-recheck builds instead of skipping graph
  publication.

Proof:

- qtlc0 rebuilt successfully with the binary graph codec;
- the cache resealed as `qtlc-checked-source-module-graph-v2`;
- direct binary validation matched 459 nodes, 1,712 edges, and 20,999
  visibility facts;
- direct graph hydration reported
  `live-module-graph-binary-materialized`;
- module-graph hydration dropped from about `2.60–3.24 s` to `0.128–0.148 s`;
- complete partial hydration dropped from about `3.27–4.06 s` to
  `0.96–1.05 s`;
- a one-file delta still rechecked 3 modules, lowered 1 unit, and restored 458
  units and objects;
- artifact profiling proved side artifacts, changed-object materialization,
  and linking together take under `0.51 s`; the displayed
  `artifact-materialize 11/11` heartbeat was carrying elapsed time from
  earlier backend work.

Finding:

- the largest remaining delta-build gap is now between frontend completion
  and one-unit lowering: approximately 10 seconds are spent probing and
  reconstructing backend restore state;
- the changed-module recheck remains approximately `2.4–2.5 s`;
- artifact materialization is not the current bottleneck.

Next:

- `POST-QTLC0-BACKEND-RESTORE-PROFILE-009` should measure native product-cache
  probing, unit-index decoding, lowered-shard index decoding, restore-plan
  construction, bundle validation, and metadata-closure reconstruction
  separately, then remove the dominant full-set walk.

## Locked Impl And View Method Contract

QuantumLang uses the implementation boundary, not a required method-level
`mut` marker, to describe receiver permissions.

```qn
impl AstArena {
  pub add(node: AstNode) {
    records.push(node)
    recordCount = recordCount + 1
  }
}

impl view AstArena {
  pub count: i64 => recordCount

  pub nodeAt(index: i64) -> AstNode {
    records[index]
  }
}
```

Locked rules:

- `impl Type` owns constructors, factories, and mutation-capable methods;
- methods in normal `impl` may mutate the receiver without a `mut` keyword;
- no declared return type means canonical `void`;
- `impl view Type` owns read-only properties and parameterized read-only
  methods;
- view methods may read fields, accept parameters, create locals, call other
  view methods, and return primitive, generic, custom, `Option<T>`, or
  `Result<T,E>` values;
- view methods may not mutate receiver state, call mutation-capable receiver
  methods, return mutable aliases, or construct/replace `this`;
- nested mutation such as `arena.add(node)` must update the real nested field
  directly, with no copy-discard behavior and no dynamic dispatch;
- value-returning APIs such as `adding(node) -> This` remain valid, but callers
  must consume their returned value.

### POST-QTLC0-IMPL-VIEW-READ-METHODS-001

Status: complete.

Contract:

- parse parameterized methods inside `impl view`;
- register and export them as receiver-bound read-only methods;
- type-check all return types, including custom and generic carriers;
- reject field assignment, mutating calls, mutable aliases, and `this`
  replacement inside view methods;
- reject constructors and every receiver-producing factory form inside
  `impl view`, including inferred returns, `-> This`, and explicit owner
  return types;
- support direct and cross-module calls such as `arena.nodeAt(index)`;
- lower view calls to static native calls or inline field/index operations;
- add positive and negative qtlc0 proofs.

Negative proof:

- `qtlc/tests/proofs/impl_view_boundary_negative.sh`
- constructors inside `impl view` must fail;
- factories returning `This` must fail;
- factories returning the explicit owner type must fail;
- inferred factories producing `This { ... }` must fail.
- calls from `impl view` into mutation-capable receiver methods must fail.

Proof:

- `impl_view_read_method_ok.qn` builds and runs with a view method calling
  another parameterized view method;
- `impl_view_boundary_negative.sh` rejects constructors, inferred and
  explicit factories, and mutation-capable calls inside `impl view`.

### POST-QTLC0-NESTED-RECEIVER-MUTATION-002

Status: complete.

Contract:

- make normal `impl` methods mutation-capable without requiring `mut`;
- propagate `arena.add(node)` through the real nested receiver;
- support arbitrary nested paths and generic/custom receiver fields;
- preserve ownership, borrow, const, and pointer authority checks;
- lower nested mutation to direct native address/store operations;
- reject mutation through `impl view`, shared references, and const pointers;
- remove qtlc1 copy/update workarounds after proof succeeds.

Proof:

- `nested_receiver_mutation_ok.qn` builds and runs;
- nested `Vec<T>` storage is fully initialized;
- `rows.add(value)` updates the real nested receiver and the matching view
  read returns the inserted value.

### POST-QTLC1-IMPL-VIEW-SOURCE-MIGRATION-003

Status: complete.

Contract:

- remove method-level `mut` from normal qtlc1 `impl` blocks;
- place guaranteed read-only receiver methods in `impl view`;
- restore direct nested mutation such as `arena.add(node)`;
- remove redundant `adding(...)` workarounds where no immutable API is needed;
- keep intentional immutable update APIs;
- build qtlc0 proofs and qtlc1;
- require qtlc1 `check` to produce nonzero AST nodes and qtlc1 `build` to run
  without a core dump.

Proof:

- qtlc1 source contains no method-level `mut` declarations;
- obsolete `adding`, `addingRoot`, and `addingDiagnostic` workarounds are
  removed from the compiler source;
- qtlc1 builds successfully;
- qtlc1 `check qtlang/compiler` reports 87,738 AST and typed nodes;
- qtlc1 `build qtlang/compiler` completes and writes `build/debug/qtlc2`
  without a core dump.

### POST-QTLC0-QUANTUM-STYLE-AGGREGATE-BACKEND-004

Status: complete.

Contract:

- preserve declared field types for positional, named, and mixed aggregate
  construction;
- treat known enum and aggregate types as concrete types, not unresolved
  generic parameters;
- resolve enum shorthand fields through the enum layout and emit the declared
  variant tag;
- lower nested target-typed default literals with the containing field type;
- store narrow fields with their real native width so `bool`, byte, and
  narrow integer fields cannot overwrite adjacent fields or stack state;
- apply the same concrete-type classification to aggregate construction and
  field access paths.

Proof:

- `qtlang/qtlc0_test_build` builds and runs successfully;
- mixed enum shorthand such as `This { symbol, Comparison, "equality" }`
  preserves the `Comparison` tag;
- nested target-typed defaults build without unsupported-expression errors;
- generic OOP surface proofs no longer corrupt caller stack state;
- qtlc1 rebuilds, checks 459 files, and builds qtlc2 without a core dump.

### POST-QTLC0-LET-MUTABLE-CONST-IMMUTABLE-001

Status: complete.

Locked rule:

```qn
let index: i64 = 0
index = index + 1

const limit: i64 = 10
```

- `let` declares a mutable local by default;
- `const` declares an immutable local and requires an initializer;
- assigning to a `const` local reports `QLTYPE0002`;
- `let mut` remains accepted as migration compatibility syntax;
- qtlc1 source uses canonical `let` without `mut`;
- IDE semantic tokens mark `let` locals mutable and `const` locals readonly.

Proof:

- `qtlang/qtlc0_test_build` mutates scalar and aggregate locals and runs;
- negative fixtures reject `const` assignment and missing initialization;
- qtlc1 builds and checks all 460 compiler source files.

## Locked Core Communication Model

```qn
(A...) -> R
event<T>
operation<T, R>
stream<T>
channel<T>
signal<T>
sink<T>
```

Locked semantics:

- `(A...) -> R`: one directly callable value;
- `event<T>`: one occurrence delivered to zero or many listeners, with no
  returned value;
- `operation<T,R>`: one typed input produces exactly one typed result, without
  assuming HTTP, IPC, local calls, plugins, devices, or another transport;
- `stream<T>`: ordered values with completion, failure, cancellation, and
  backpressure lifecycle;
- `channel<T>`: concurrent buffered transfer with ownership, send/receive,
  ordering, capacity, and backpressure;
- `signal<T>`: readable current state plus change observation;
- `sink<T>`: write-only value destination.

These are compiler-known core forms and require no user import. Runtime
scheduling and transport implementations remain separate toolchain/runtime
components.

### POST-QTLC0-QUANTUM-CALLABLE-TYPE-SYNTAX-001

Status: complete.

Contract:

- accept `(A, B) -> R` as the canonical Quantum source spelling;
- normalize it internally to the existing native callable representation;
- support zero, one, many, generic, custom, `Option<T>`, and `Result<T,E>`
  parameter and result types;
- preserve direct native function-pointer calls with no reflection, boxing, or
  dynamic name lookup;
- keep legacy `fn(A, B) -> R` as bootstrap compatibility syntax only.

### POST-QTLC1-CORE-COMMUNICATION-MODEL-002

Status: complete.

Contract:

- define distinct `event`, `operation`, `stream`, `channel`, `signal`, and
  `sink` core contracts;
- keep policy configuration typed and compile-time visible;
- model delivery, wait, overflow, capacity, lifecycle, ownership, and
  backpressure without collapsing these forms into generic callbacks;
- add qtlc1 source proof using all seven forms;
- keep runtime execution out of qtlc1 seed scope.

### POST-QTLC0-CORE-COMMUNICATION-NATIVE-PROOF-003

Status: complete.

Contract:

- prove callable fields lower to direct native callable slots;
- prove known direct calls inline or lower to direct calls;
- prove communication policy views use direct field loads;
- prove no reflection, string dispatch, hidden map, or mandatory allocation;
- add negative proofs for invalid callable arguments and incompatible result
  types.

Proof:

- callable values lower to native callable slots and indirect native calls;
- policy and channel views lower to direct native view functions;
- the owned proof assembly contains no allocation, reflection, string
  dispatch, or hidden map lookup;
- invalid callable argument and result signatures fail with `QLTYPE0003`;
- named function identifiers now infer their complete callable signature;
- `qtlc check` success artifacts include and validate the running compiler
  identity, so stale success cannot hide changed type-checker behavior.

## Communication, Parallel, And Real Compiler Sequence

This order is locked. qtlc1 owns language contracts, checking, HIR/MIR, and
native zero-cost proofs. qtlc2/qtlc3 own production scheduling and transport
runtime implementations.

### POST-QTLC1-COMMUNICATION-API-CONTRACT-004

Status: complete.

Contract:

- complete typed APIs for `event<T>`, `operation<T,R>`, `stream<T>`,
  `channel<T>`, `signal<T>`, and `sink<T>`;
- define typed lifecycle, errors, cancellation, subscription tokens, ordering,
  capacity, and backpressure behavior;
- support empty/default construction and target-typed literals;
- keep direct callable syntax `(A...) -> R`;
- do not fake queues, waiting, wake-up, or transport execution inside qtlc1.

Implemented:

- typed communication lifecycle, error kinds, and subscription tokens;
- typed ordering and ownership-transfer policy;
- `event.subscribe/unsubscribe/emit/close/cancel`;
- `operation.bind/call/close/cancel`;
- `stream.push/next/tryNext/complete/fail/cancel`;
- `channel.send/trySend/receive/tryReceive/close/cancel`;
- `signal.read/set/observe/unobserve/close`;
- `sink.write/tryWrite/close/cancel`;
- bounded, parallel, and realtime policy constructors;
- qtlc0-checked default and target-typed construction.

### POST-QTLC0-COMMUNICATION-HOT-PATH-005

Status: complete.

Contract:

- lower callable slots and known calls directly;
- inline policy/lifecycle/readiness views to direct native loads;
- prove no reflection, string dispatch, hidden map, or mandatory allocation;
- add executable, assembly, and negative type proofs.

Verification:

- `communication_hot_path_ok.qn` builds and executes successfully;
- `communication_hot_path_asm.sh` proves direct callable dispatch and direct
  communication views;
- the assembly proof reports
  `allocation=false reflection=false hidden_map=false`;
- callable argument and callable result mismatch fixtures are rejected by
  `qtlc check`.

### POST-QTLC1-PARALLEL-LOOP-CONTRACT-001

Status: complete.

Contract:

- model `par for` and `shard for`;
- enforce capture, ownership, shard-local mutation, deterministic reduction,
  implicit join, cancellation, and backpressure facts;
- assign stable body, partition, and join IDs;
- keep scheduler execution outside qtlc1.

Implemented:

- distinct `ParFor` and `ShardFor` loop contracts;
- read, move, shard-local, and rejected shared-mutable capture facts;
- ordered and associative reduction policy;
- stable region, body, partition, and join IDs;
- implicit join, cancellation, backpressure, partition, and deterministic
  reduction validation;
- qtlc1 proof integrated with the existing deferred parallel-region proof;
- full 470-file compiler package check passes.

### POST-QTLC1-PARALLEL-HIR-MIR-002

Status: complete.

Contract:

- lower parallel loops to typed HIR/MIR partition and join regions;
- represent communication edges with channel/stream/sink contracts;
- produce stable incremental query products;
- emit target-neutral scheduler requirements, not a runtime implementation.

Implemented:

- typed HIR parallel regions for `par for` and `shard for`;
- explicit HIR capture, reduction, partition, join, cancellation, and
  backpressure facts;
- typed MIR partition, body, and join regions;
- channel, stream, and sink communication edges between MIR regions;
- target-neutral scheduler requirements for partition count, deterministic
  execution, shard-local mutation, ordered reduction, implicit join,
  cancellation, and backpressure;
- stable HIR and MIR incremental keys derived from region, body, partition,
  and join IDs;
- locked `par for` for independent work and `shard for` for shard-local
  mutation with deterministic merge;
- explicit `parallel N { ... }` worker-region contract with sequential
  `while` and `loop` bodies inside each worker;
- typed `atomic<T>` shared-state facts with sequential consistency by default
  and invalid load/store memory-order rejection;
- normal shared mutable capture remains a compile error;
- no separate `par while` or `par loop` implementation;
- no scheduler, queue, worker, or transport runtime implementation in qtlc1.

Verification:

- qtlc0 checks all 475 qtlc1 compiler source files;
- qtlc0 builds the native qtlc1 executable successfully;
- the rebuilt qtlc1 checks all 475 compiler source files successfully;
- the parallel HIR/MIR proof validates typed communication edges, stable
  product keys, target-neutral scheduler requirements, worker-region facts,
  atomic shared-state facts, and invalid memory-order rejection;
- the central qtlc1 type catalog recognizes lowercase `atomic<T>` as a
  target-gated concurrency type.

### POST-QTLC1-REAL-MODULE-RESOLUTION-004

Status: implemented.

Contract:

- build the real module graph from concrete declaration records;
- resolve imports, exports, aliases, visibility, and duplicates;
- produce structured name-resolution diagnostics.

Implemented:

- concrete `ModuleNode`, `ModuleEdge`, `ImportPath`, and declaration-record
  ownership in `ModuleGraph`;
- import alias indexing and alias-conflict detection;
- public export lookup with private declarations excluded;
- duplicate module/declaration and unresolved-import diagnostics;
- resolver scopes and symbols populated from concrete declaration records;
- cross-module module-resolution proof covering resolved imports, aliases,
  exports, hidden private declarations, symbols, and diagnostics.

qtlc0 capability fixes required without weakening qtlc1:

- block methods without an explicit return type now remain `void`; only
  explicit-return blocks, expression bodies, view properties, and constructor
  results receive implicit returns;
- native overload dispatch preserves concrete generic parameter types when
  definitions are uniquified, so `Vec<DeclarationRecord>` overloads link to
  their real body symbols.

Verification:

- focused void aggregate-mutation fixture builds and runs natively;
- focused generic aggregate-overload fixture builds and runs natively;
- qtlc0 checks all 473 qtlc1 compiler source files;
- qtlc0 builds and links the native qtlc1 executable;
- the rebuilt qtlc1 checks all 473 compiler source files successfully.

### POST-QTLC1-REAL-TYPECHECK-005

Status: in progress after real module resolution.

Contract:

- build real symbol and type environments;
- type expressions, calls, generics, carriers, communication forms, and
  parallel regions;
- produce structured type diagnostics and typed products for HIR lowering.

Implemented first slice:

- `TypeSymbol` now preserves concrete declaration ID, module, visibility, and
  exact declaration kind;
- `TypeSymbolTable` is built from concrete parser-produced
  `DeclarationRecord` values instead of frontend summary counts;
- module/type/role/behavior/import/public/private counts are derived from the
  concrete symbol rows;
- direct module/name and declaration-ID lookup APIs are available for later
  expression and call typing;
- `TypeEnvironment` now consumes the record-backed symbol table.

Remaining:

- complete member ownership lookup and full parser expression-tree precedence;
- infer generic substitutions and overload ranking beyond exact arity;
- replace temporary unknown callable results after every callable declaration
  carries a concrete return signature.

### POST-QTLC1-CLI-REAL-PROGRESS-002

Status: implemented.

Contract:

- report completed work from the real frontend, symbol, scope, and typed-node
  loops;
- never display a fabricated `0/N` phase;
- show the currently processed module;
- refresh TTY output at 100 ms and non-TTY output at one second;
- emit immediately when the phase or diagnostic count changes;
- keep `--quiet`, `--verbose`, color policy, `NO_COLOR`, and cancellation
  behavior centralized in the terminal renderer;
- avoid allocation per AST node.

Implementation:

- `platform.CompilerProgress` is the dependency-neutral compiler-work bridge;
- frontend progress advances once after read, lex, and parse for each module;
- symbol, scope, and type loops publish primitive counters every 256 rows and
  at completion;
- the process-wide native progress state owns throttling, TTY animation,
  stable non-TTY lines, diagnostics, and verbose token/symbol/scope counters;
- driver session code no longer publishes estimated zero-completion events.

Output model:

```text
[Frontend] 987/1647 59%  qtlang/compiler/syntax/parser/sourceParser.qn
[Typechecking] 93811/133869 70%  typeSystem::checking::typeRegistry
check OK  qtlang/compiler  0 diagnostics  14.20s
```

TTY output uses one in-place line with color when supported. Non-TTY output
uses throttled stable lines. `--quiet` suppresses progress and keeps only
diagnostics and the final result. `--verbose` adds jobs, tokens, symbols, and
scopes.

### POST-QTLC1-CENTRAL-DIAGNOSTIC-REGISTRY-001

Status: implemented for the active qtlc1 typechecking and literal pipeline.

Contract:

- stable external diagnostic strings remain compatible;
- compiler logic selects typed subsystem codes instead of constructing strings;
- one central registry owns numeric identity, category, severity, title,
  message formatting, help, and documentation paths;
- subsystem enums remain separate so the catalog does not become one giant
  global enum;
- success paths carry an empty `DiagnosticSpec`;
- diagnostic strings are materialized only at reporting and compatibility
  boundaries.

Implemented catalog:

```text
TypeDiagnosticCode      -> QTTYPE0001..QTTYPE0402
LiteralDiagnosticCode   -> QTLIT0001..QTLIT0008
FunctionDiagnosticCode  -> QTFN0001..QTFN0003
```

Implemented consumers:

- expression typing rules and resolutions;
- unresolved member values and member calls;
- typed-node failures;
- declaration-symbol and lexical-binding issues;
- literal target resolution and literal diagnostic facts;
- function return validation;
- final `TypeDiagnosticSet` rows carry structured `DiagnosticCode` values.

Compatibility views such as `.diagnosticCode` still expose stable strings for
tools and existing proofs, but those strings derive from the registry.

Proof:

- `diagnosticRegistryProof` checks stable code values, uniqueness across the
  implemented catalog, structured metadata, help/documentation ownership, and
  deterministic message formatting.

Remaining subsystem migrations:

- parser and import diagnostics;
- control-flow diagnostics;
- HIR and MIR diagnostics;
- quantum diagnostics.

Those migrations add dedicated typed code enums under `diagnostics/codes/`;
they must not add raw code strings back into compiler logic.

### POST-QTLC1-OVERLOAD-GENERIC-DESIGN-AUDIT-012

Status: implemented.

The resolver contract is locked in
[QTLC1_OVERLOAD_GENERIC_RESOLUTION_DESIGN.md](QTLC1_OVERLOAD_GENERIC_RESOLUTION_DESIGN.md).

Audit coverage:

- symbol and scope indexes;
- structural auto-constructors and custom constructors;
- roles, concrete extensions, and bounded blanket extensions;
- callable values and callable types;
- contextual and builtin operator forms;
- central diagnostic ownership;
- generic inference, conversion, ranking, cache, typed-node, and HIR
  contracts.

The audit found that current lookup is primarily name/arity based, call AST
rows preserve counts rather than structured arguments, and owner generic
specialization still uses limited string substitution. These are explicit
replacement targets for `...-013`, not accepted final behavior.

Locked rules:

- candidate collection is indexed and returns a candidate set;
- argument binding is positional/named/default/variadic aware;
- generic inference is structural over `TypeId`;
- ranking uses a deterministic lexicographic cost vector;
- declaration order and stable IDs never select a winner;
- equal viable costs produce ambiguity;
- overload resolution remains compile-time and reflection-free;
- MIR receives a committed call plan and performs no overload search.

### POST-QTLC1-REAL-TYPECHECK-OVERLOAD-GENERIC-013

Status: next.

Implementation must follow the audited `013A` through `013J` order in
[QTLC1_OVERLOAD_GENERIC_RESOLUTION_DESIGN.md](QTLC1_OVERLOAD_GENERIC_RESOLUTION_DESIGN.md).

The implementation includes:

- structured parameter, generic parameter, and call argument rows;
- candidate-set indexes for free calls, members, constructors, roles,
  extensions, callable values, and operators;
- positional, named, defaulted, and variadic argument binding;
- structural generic inference and bound validation;
- typed conversion facts and deterministic ranking;
- specific overload/generic diagnostics;
- committed typed-node/HIR call plans;
- resolution caching and precise invalidation proofs.

#### POST-QTLC1-REAL-TYPECHECK-OVERLOAD-GENERIC-013A

Status: implemented.

- added parser-owned `AstCallArgument`, `AstCallableParameter`, and
  `AstGenericParameter` rows;
- call arguments preserve position, optional name, expression text, inferred
  type, source span, named form, and spread form;
- callable parameters preserve position, name, declared type, default
  expression, source span, default state, and variadic state;
- generic parameters preserve position, name, bounds, default type, source
  span, and variadic state;
- `AstDeclaration`, `DeclarationRecord`, and `TypeSymbol` carry the structured
  rows unchanged;
- exact-arity symbol indexes consume structured callable parameter counts when
  available;
- compatibility name vectors remain temporarily for existing qtlc1 code, but
  are no longer the intended resolver source of truth;
- syntax, declaration, and type-symbol stable hashes include the structured
  rows.

Proof:

- `overloadGenericRowsProof` parses bounded/defaulted generics,
  required/defaulted/variadic parameters, and positional/named call arguments,
  then verifies frontend and type-symbol preservation.

Next:

#### POST-QTLC1-CANDIDATE-INDEX-DESIGN-AUDIT-013B0

Status: implemented.

The audit is locked in
[QTLC1_CANDIDATE_INDEX_DESIGN_AUDIT.md](QTLC1_CANDIDATE_INDEX_DESIGN_AUDIT.md).

Findings:

- name and scope lookup indexes currently collapse repeated keys to one row;
- free-call/member-call APIs return one symbol and may select by declaration
  order;
- generic member and free-call fallbacks still scan the full symbol table;
- lexical unique bindings and callable overload buckets need separate index
  contracts;
- constructors are not yet represented as structural/custom candidate rows;
- builtin operators bypass the unified candidate pipeline;
- qtlc1 syntax/frontend records do not yet preserve locked Design-10
  per-conformance `pub/internal/protected/private` visibility.

Locked `013B` contract:

- one centralized `CandidateIndex` references canonical `TypeSymbolTable`
  symbols by stable ID;
- free, lexical, member, constructor, extension, and operator keys return
  stable unranked `CandidateSet` slices;
- candidate rows retain source, access, owner, callable type, generic shape,
  defaults, and variadic facts;
- arity is a secondary bucket filter, never the overload identity;
- candidate collection performs no argument binding, generic inference,
  ranking, or HIR commitment;
- lookup target is average `O(1) + O(k)` with no per-call full-symbol scan;
- invalidation is per affected bucket and includes role visibility and module
  export identity.

Next:

- `POST-QTLC1-REAL-TYPECHECK-OVERLOAD-GENERIC-013B`: implement the audited
  candidate-set indexes and switch expression call lookup away from
  first-symbol/full-scan behavior.

#### POST-QTLC1-REAL-TYPECHECK-OVERLOAD-GENERIC-013B

Status: implemented candidate-collection slice.

- added centralized free, member, constructor, extension, and operator
  candidate buckets;
- candidate rows retain canonical symbol row/ID, callable type, source,
  access, generic arity, defaults, variadic shape, and declaration span;
- expression call lookup no longer selects the first module declaration or
  performs a full-symbol fallback scan;
- only one arity-compatible candidate may be committed before ranking exists;
- role-method visibility is indexed per exact role method and preserves
  `pub`, `internal`, `protected`, and `private`;
- `protected` role visibility is supported by qtlc0 and qtlc1 and remains
  unavailable to unrelated callers;
- structural constructors and builtin binary operators enter the same
  candidate boundary;
- qtlc0 native role visibility proof builds and runs;
- full `qtlang/compiler` check passes.

Next:

#### POST-QTLC1-REAL-TYPECHECK-OVERLOAD-GENERIC-013C

Status: implemented first ranked call-resolution slice.

- binds positional, named, mixed, defaulted, and variadic arguments;
- preserves explicit generic argument names and infers simple direct and
  single-container generic substitutions;
- validates indexed role bounds and generic defaults;
- computes exact, alias, numeric-widening, deferred, default, generic,
  variadic, and candidate-source costs;
- selects the unique lexicographically best viable candidate;
- reports deterministic ambiguity, generic mismatch, arity, duplicate,
  unknown-name, access, bound, and conversion rejection facts;
- integrates ranked free, member, and constructor resolution into expression
  typing;
- adds direct executable proof coverage for exact overloads, defaults,
  variadics, inference, explicit generics, and ambiguity;
- qtlc0 native associated-method lookup now falls back from the fast host
  index to full semantic impl metadata for private same-owner methods;
- full `qtlang/compiler` source check passes with 577 files.

Native proof gate:

- qtlc1 package API artifact emission is currently blocked by the existing
  qtlc0 contextual unary-operator lowering gap across qtlc1 sources;
- the focused dependency proof package is ready and should run after qtlc1
  `.qcap/.qidx/.qna` artifacts can be emitted.

Next:

- `POST-QTLC1-REAL-TYPECHECK-OVERLOAD-GENERIC-013D`: replace string
  substitution with structural `TypeId` unification, recursively support
  nested generic arguments, and commit constructor field binding.

### POST-QTLC1-REAL-TYPECHECK-EXPRESSION-009

Status: implemented first real expression slice.

- parser AST nodes preserve call argument count, generic argument count,
  declared local types, and callable return types;
- literals, identifiers, unary/binary operators, assignments, calls,
  constructors, carriers, collections, communication forms, parallel regions,
  and atomics produce concrete expression-resolution facts;
- exact-arity callable lookup and stable constructor/function symbol identity
  are preserved through `TypedNodeFact`;
- typed rows store stable `NodeId`, `SymbolId`, `TypeId`, scope, operator,
  target type, call arity, and generic arity;
- type diagnostics carry expression-specific codes and messages;
- `TypedHirProduct` now transfers stable type/symbol identity, operator symbol,
  and call/generic arity into real `HirExpr` rows.

Verification proof:

- `realExpressionTypecheckProof` covers literals, binary and assignment
  operators, constructor/function calls, Option, collections, communication,
  parallel/atomic typing, stable IDs, and structured invalid-operator
  diagnostics.

### POST-QTLC1-REAL-TYPECHECK-MEMBER-CALL-010

Status: implemented first owner-aware member-call slice.

- AST declarations and declaration records preserve `ownerTypeName`;
- the source parser tracks `impl`, `impl view`, `context`, `extract`, and
  `extend` ownership while producing callable declaration rows;
- call AST nodes preserve a direct receiver name for `receiver.member(...)`;
- `TypeSymbol` preserves member ownership and concrete callable return type;
- `TypeSymbolTable` owns deterministic indexed lookup by owner, member name,
  argument arity, and generic arity;
- inferred generic member calls use a separate index instead of scanning all
  symbols;
- member calls resolve to stable `SymbolId` and `TypeId` with concrete return
  types instead of `callable-result`;
- receiver type and member-call identity flow through `TypedNodeFact` into
  `HirExpr`;
- unresolved member calls produce structured `QTTYPE0104` diagnostics.

Verification:

- full `qtlc check -j8 qtlang/compiler` passes with 548 source files;
- `realExpressionTypecheckProof` covers owner-aware lookup and overload
  selection;
- the standalone `qtlang/compiler/tests` package remains blocked by existing
  import-root and unrelated hash-proof parse failures.

Remaining:

- preserve full receiver expression trees instead of only direct receiver
  names;
- resolve fields and `impl view` properties through owner layout metadata;
- apply generic substitutions to receiver and return types;
- rank compatible overloads beyond exact arity;
- feed resolved member argument types into HIR and MIR call signatures.

### POST-QTLC1-REAL-TYPECHECK-MEMBER-VALUE-011

Status: implemented first owner-aware field/property and deep receiver slice.

- shape fields and `impl view` properties are parser-produced declaration
  records with explicit owner, member category, and value type;
- methods, fields, and properties share declaration ownership without adding
  parallel declaration enums or helper wrappers;
- fields and properties become `TypeSymbolKind.Value`, so they are not counted
  as callable functions;
- owner-scoped duplicate checks allow different types to use the same field
  name while rejecting duplicate values on one owner;
- `TypeSymbolTable` owns a dedicated indexed value-member lookup by owner and
  member name;
- AST member nodes preserve complete receiver paths such as
  `user.profile.address`;
- receiver paths resolve one typed member at a time through indexed lookup,
  with no global symbol scan and no runtime reflection;
- member field/property type, stable symbol/type identity, and resolved
  receiver type flow through `TypedNodeFact` into HIR;
- unresolved member values produce structured `QTTYPE0105` diagnostics.

Verification:

- full `qtlc check -j8 qtlang/compiler` passes with 548 source files;
- `realFrontendOwnershipProof` covers parser-owned fields and deep paths;
- `realExpressionTypecheckProof` covers fields, `impl view` properties, and a
  two-level receiver chain.

Remaining:

- preserve indexed/call receiver expressions, not only identifier chains;
- apply generic substitutions to field/property and method return types;
- carry resolved field layout offsets into MIR/native lowering;
- rank overloads by argument types and conversion cost.

### POST-QTLC1-REAL-TYPECHECK-GENERIC-SUBSTITUTION-012

Status: implemented first owner-generic substitution slice.

- concrete receiver owners such as `Box<User>` match declarations owned by
  `Box<T>`;
- member field, property, and method symbols are specialized before expression
  resolution;
- direct `T` and carrier forms such as `Option<T>`, `Vec<T>`, and
  `Result<T,E>` preserve concrete receiver arguments;
- specialized symbols retain their stable declaration, symbol, and type IDs;
- exact-owner indexed lookup remains the hot path; generic-owner matching is a
  deterministic fallback.

Verification:

- `realExpressionTypecheckProof` covers `Box<T>.value -> T` and
  `Box<T>.maybe -> Option<T>` specialized for `Box<User>`.

Remaining:

- support multiple owner generic parameters;
- infer method generic arguments from typed call arguments;
- substitute nested callable and tuple types structurally;
- rank overloads by exact type, generic unification, and conversion cost.

### POST-QTLC1-TYPECHECK-SYMBOL-SCOPE-INDEX-008

Status: implemented.

- `TypeSymbolTable` owns deterministic XXH3-indexed unqualified and
  module-qualified name lookups;
- exact qualifier/name checks preserve correctness under hash collisions;
- `TypeScopeTable` owns an indexed `(ScopeId, name)` binding lookup;
- bucket counts and rows use `usize`; collision links use `Option<usize>`
  instead of signed `-1` sentinels;
- collision chains use `while let Some(entry) = current` so Option traversal
  remains concise and lowers to the direct tagged-value loop;
- lexical parent traversal remains explicit, but each scope probe is average
  O(1) instead of rescanning every binding;
- duplicate symbol and duplicate binding diagnostics use the same indexes;
- stable `SymbolId` and `TypeId` records remain the semantic identity, never
  pointer addresses or allocation order.

This removes the repeated symbol-table and scope-binding scans from the
per-AST-node typecheck path.

Verification:

- qtlc0 checks the control-flow and type-system modules successfully;
- qtlc0 checks all 475 qtlc1 compiler source files successfully;
- qtlc0 builds the native qtlc1 executable;
- the rebuilt qtlc1 checks all 475 compiler source files successfully.

### POST-QTLC0-CENTRAL-OPERATOR-REGISTRY-001

Status: implemented.

- qtlc0 owns one canonical operator registry for IDs, symbols, groups,
  precedence, associativity, arity, mutation, assignment bases, and role
  dispatch names;
- lexer token identity, parser precedence, assignment parsing, type checking,
  CTFE rendering, and native/C/LLVM assignment lowering consume that registry;
- `impl view` permits local mutation but rejects receiver mutation using the
  registry mutation fact;
- qtlc1 `BuiltinOperatorFact` mirrors the current qtlc0 operator contract;
- `operator_registry_groups_pkg` is the native executable proof for
  arithmetic, power, increment, bitwise, shift, assignment, comparison,
  logical, range, access, conversion, and impl-view local mutation.

### POST-QTLC0-CONTEXTUAL-OPERATOR-FORMS-002

Status: implemented.

- `OperatorId` identifies semantic operations while `OperatorForm` identifies
  binary, prefix, postfix, assignment, reference, move, dereference, call,
  index, path, propagation, optional-type, spread, and syntax contexts;
- shared tokens are no longer semantically collapsed: `*` distinguishes
  multiply/dereference, `&` distinguishes bit-and/reference, `^` distinguishes
  xor/move, and `?` distinguishes propagation/optional type;
- parser expression/type/generic paths query the contextual registry;
- qtlc1 `BuiltinOperatorFact.fromForm(...)` mirrors contextual forms;
- qtlc1 proofs now match qtlc0: `..` is half-open, `..=` is inclusive,
  `++`/`--` are valid, and `??`/`..<` remain unsupported.

### qtlc2/qtlc3 Production Runtime

Status: deferred until qtlc1 is a real self-hosted seed.

Owns:

- work-stealing scheduling;
- execution of `parallel N`, `par for`, and `shard for` regions;
- atomic load/store/swap/compare-exchange and fetch operations;
- target-specific memory-order lowering and atomic-width capability checks;
- deterministic shard reduction and implicit join execution;
- lock-free queues;
- cancellation and wake-up execution;
- backpressure execution;
- NUMA, affinity, GPU, and device scheduling;
- kernel-bypass implementations supplied through platform/provider plugins.

Planned commands:

```text
POST-QTLC2-PARALLEL-RUNTIME-001
  execute parallel worker, par-for, shard-for, cancellation, and join regions

POST-QTLC2-ATOMIC-RUNTIME-002
  implement scalar atomic operations, ordering, wait/wake, and target gates

POST-QTLC2-PARALLEL-SCHEDULER-003
  add work stealing, bounded queues, backpressure, and deterministic reduction

POST-QTLC3-PARALLEL-TARGETS-004
  add NUMA, affinity, GPU/device, quantum-provider, and kernel-bypass plugins
```

### POST-QTLC0-COMPONENT-IDENTITY-PER-UNIT-RESTORE-001

Status: implemented; final object/relink delta proof is blocked by current
qtlc1 native ABI diagnostics.

- cache identity is separated into frontend, semantic, lowering, emission,
  and linker components;
- backend-only qtlc0 changes no longer invalidate checked-source/frontend
  state;
- lowered-unit metadata stores and compares a per-unit ABI hash instead of
  rejecting the full package when the package ABI aggregate changes;
- lowered units are restored when the qtlc0 executable changes but the
  lowering component identity remains stable;
- lowered shards reference one `shared.meta` file for version, lowering
  identity, and semantic identity;
- checked-source products are sealed before native emission, so emit/link
  failure preserves reusable frontend state;
- per-unit object keys use emission identity while executable link cache uses
  linker identity.

Observed proof after a native backend failure:

```text
checked-source full-typed-set: hydrate=488 recheck=0
builder skipped: parse=0 check=0
native backend unit cache: unchanged=488 changed=0 missing=0
lowered payload restore: restored=488 lower_plan_units=0
backend lowering: units=0 restored_units=488
```

Observed one-file semantic delta proof:

```text
frontend: hydrate=24 recheck=1 parse=1
module impact: roots=1 affected=1
backend unit cache: unchanged=24 changed=1 missing=0
metadata dependency closure: 1 unit
native partial emit: changed_inputs=1 metadata_inputs=1
objects: restored=24 compiled=1 failed=0
summary: lowered=1 emitted=2 linked=1
```

The qtlc1 package itself still has unrelated native ABI diagnostics in
progress/name-resolution code. Those diagnostics block a successful qtlc1
link, but no longer discard its reusable frontend or lowered-unit state.

## Modern Option Surface

```text
POST-QTLC0-OPTION-MODERN-HOT-PATH-001
  status: scalar carrier surface implemented
  Option<T>.present and .missing are read-only tag views
  valueOr(default) is the canonical checked fallback operation
  map/then/filter/or/orCreate/require/matches are statically typed combinators
  unchecked Option<T>.value is rejected
  no reflection, allocation, named-argument map, or dynamic dispatch
  pointer/reference payloads may use a one-word niche representation
  remaining qtlc0 blocker: native aggregate Option<T> payload execution
  callback-combinator execution: POST-QTLC0-OPTION-COMBINATOR-NATIVE-002
```

Locked source surface:

```qn
option.present
option.missing
option.valueOr(defaultValue)
option.map(value => convert(value))
option.then(value => next(value))
option.filter(value => accept(value))
option.or(other)
option.orCreate(() => createValue())
option.matches(value => accept(value))
option.require(Error.NotFound)
```

`present` and `missing` lower to one carrier tag test. `valueOr` lowers to a
branch/select. Callback combinators must be statically specialized and
inlineable; metadata-only or dynamically dispatched implementations do not
satisfy this command.

```text
POST-QTLC0-OPTION-COMBINATOR-NATIVE-002
  status: implemented for qtlc0 native Option carriers
  map/then/filter/matches use direct tag branches
  inline callbacks are statically substituted
  named callbacks resolve to direct native symbols
  orCreate evaluates its callback only in the None branch
  or/require/valueOr operate directly on carrier slots
  no combinator runtime helper, allocation, reflection, or indirect dispatch
  proof: qtlc/tests/proofs/option_modern_combinator_hot_path_asm.sh
```

Aggregate payload layout and pointer/reference niche representation remain
separate roadmap items. They do not change this combinator control-flow
contract.

```text
POST-QTLC0-OPTION-AGGREGATE-LAYOUT-003
  status: implemented for qtlc0 native Option carriers
  Option<User> uses a pointer-backed aggregate payload and direct field copy
  Option<Vec<T>> uses one pointer word for the native collection header
  None zeroes only the carrier word, never adjacent inline collection storage
  nested Option payloads use a deterministic flattened tag/payload layout
  Option<T> and Result<T,E> inherit copy, move-only, and drop behavior
  explicit move disables the source drop and schedules one destination drop
  invalid post-move access is rejected with QLOWN0001
  proof package: qtlang/qtlc0_option_aggregate_proof
  ownership proof: qtlc/tests/fixtures/option_aggregate_use_after_move_bad.qn
  assembly proof: qtlc/tests/proofs/option_aggregate_layout_hot_path_asm.sh
```

```text
POST-QTLC0-OPTION-CENTRAL-PAYLOAD-REGISTRY-004
  status: implemented in qtlc0 type infrastructure
  one recursive registry owns Option/Result payload kind and carrier width
  copy, move-only, and drop behavior derive recursively from payload children
  primitive, aggregate, pointer, collection, view, and nested carriers share it
  Map/Slice/Index/Arena/Array/Range carriers use deterministic native layouts
  unsized, recursive-by-value, opaque, callable, and unresolved layouts reject
  ownership, executable validation, backend metadata, and native emit consume it
  proof package: qtlang/qtlc0_option_aggregate_proof
```
## Core Text

```text
POST-QTLC1-CORE-TEXT-CATALOG-001
  grouped complete qtlc1 text method catalog
  canonical names plus compatibility aliases
  import-free core separated from codec/security extensions
  native readiness and allocation facts

POST-QTLC1-CORE-TEXT-LITERAL-FORMAT-002
  status: implemented
  strict literal escape validation
  raw and multiline literal support
  static TextFormatPlan with compile-time format-spec parsing

POST-QTLC0-TEXT-CANONICAL-ALIASES-003
  native canonical aliases for byteLength, charLength,
  graphemeLength, and substring

POST-QTLC0-TEXT-REPLACE-NATIVE-004
  real replace kernel, ABI, lowering, and executable proof
```
## POST-QTLC0-RANGE-LITERAL-FULL-MODEL-001

Status: source/type model implemented; final native self-host verification pending.

- Canonical exclusive range: `a..<b`.
- Inclusive range: `a..=b`.
- Open and full ranges: `a..`, `..b`, `..=b`, `..`.
- Finite `Range<T>` is exactly `{ start, count }` with a 16-byte native ABI.
- Open forms are `RangeFrom<T>`, `RangeTo<T>`, `RangeToInclusive<T>`, and
  `RangeFull`.
- `RangeBound<T>` describes an individual bound; `RangeSpec<T>` resolves any
  range form against a collection length.
- Range loops, patterns, and zero-allocation borrowed slicing.
- `a..b` remains a compatibility spelling and is not the preferred style.

Verified:

- qtlc0 builds successfully.
- `qtlc/tests/proofs/range_literal_hot_path_asm.sh` passes.
- `qtlc/tests/fixtures/range_literal_full_model_ok.qn` passes.
- The complete 578-file `qtlang/compiler` source check passes.

Current blocker:

- The qtlc1 native build reaches emission but qtlc0's native feature gate
  rejects `Range<T>` methods and computed views in `core/range.qn`.
- This is a qtlc0 backend capability gap. Do not remove Range behavior or
  weaken qtlc1 source to bypass it.

## POST-QTLC0-RANGE-NATIVE-FEATURE-GATE-003

Status: next.

- Recognize canonical `Range<T>` as the two-word native aggregate
  `{ start: i64, count: i64 }`.
- Permit direct native lowering for `contains`, `containsRange`, `overlaps`,
  `before`, `after`, `shift`, `grow`, `growTo`, `take`, `drop`, and
  `intersection`.
- Permit direct computed views for `end`, `empty`, `nonEmpty`,
  `hasValidStart`, `hasValidCount`, `hasValidEnd`, and `valid`.
- Inline nested Range views such as `valid`, `other.valid`, and `other.end`.
- Emit direct loads, comparisons, arithmetic, branches, and two-word copies.
- No allocation, reflection, dynamic dispatch, helper ABI, or generated C/C++.
- Keep unsupported unbounded iteration rejected at compile time.

## POST-QTLC0-RANGE-ABI-CONTRACT-PROOF-004

Status: pending after `...FEATURE-GATE-003`.

- Add positive executable proof for every finite Range method and view.
- Add assembly proof for offsets `start@0`, `count@8`, total size `16`.
- Prove bounded loops lower to compare/increment control flow.
- Prove slicing resolves open specifications to a finite Range without
  allocation.
- Add negative proof that a declared Range layout with extra fields, wrong
  offsets, or a size other than 16 is rejected.
- Add negative proof that `RangeFrom`, `RangeTo`, `RangeToInclusive`, and
  `RangeFull` cannot drive a finite loop before resolution.

## POST-QTLC1-RANGE-MODEL-INTEGRATION-005

Status: source changes implemented; native verification pending.

- Keep `Range<T>` finite and compact.
- Keep `RangeFrom<T>`, `RangeTo<T>`, `RangeToInclusive<T>`, and `RangeFull`
  as distinct open literal types.
- Keep `RangeBound<T>` as an individual-bound descriptor only.
- Keep `RangeSpec<T>.resolve(length)` as the checked conversion to finite
  `Range<T>`.
- Preserve explicit `usize` to `i64` collection-length validation through
  `RangeError.LengthOverflow`.
- Keep the qtlc1 core catalog and literal metadata synchronized with every
  range family.

## POST-QTLC1-RANGE-SELFHOST-VERIFY-006

Status: pending.

Run in this order:

```text
cmake --build qtlc/build --target qtlc -j8
bash qtlc/tests/proofs/range_literal_hot_path_asm.sh
qtlc/build/compiler/driver/qtlc check qtlc/tests/fixtures/range_literal_full_model_ok.qn
qtlc/build/compiler/driver/qtlc check -j8 qtlang/compiler
qtlc/build/compiler/driver/qtlc build -j8 qtlang/compiler
qtlang/compiler/build/debug/qtlc1 check -j8 qtlang/compiler
```

Success requires:

- qtlc1 links successfully.
- qtlc1 check does not crash.
- No undefined Range symbols.
- No native feature-gate diagnostics from `core/range.qn`.
- The finite Range ABI remains exactly 16 bytes.
- Open-range types never enter the finite Range ABI before resolution.

## POST-QTLC0-REMOVE-SELF-OWNER-ALIAS-001

Status: implemented.

- `This` is the only owner-type and fresh-construction keyword.
- `this` remains the current instance for field access and copy/update.
- Impl receivers are implicit; methods never declare a `self`/`this` receiver
  parameter.
- Bare same-impl calls remain valid when resolution is unambiguous.
- Source-level `Self`, `Self.method(...)`, `-> Self`, and lowercase `self`
  receiver access are rejected with targeted diagnostics.
- Parser-generated receiver types, generic substitutions, role metadata,
  semantic signatures, lowering, and native emission use canonical `This`.
- No active qtlc1 source or qtlc0 compiler path retains the capital `Self`
  alias.
- Positive proof: `qtlang/qtlc0_test_build/src/staticImplCallProof.qn`.
- Negative proof: `qtlc/tests/proofs/owner_alias_negative.sh`.
POST-QTLC0-RANGEBOUND-GENERIC-NATIVE-LOWERING-002
- Preserve qualified generic associated calls in semantic metadata.
- Infer generic associated owner arguments from explicit receivers, values, or target types.
- Lower RangeBound included/excluded/unbounded with native enum and Option payloads.
- Prefer real aggregate layouts over legacy collection-size shortcuts.
- Select Arena(count) by exact arity and preserve deep stableKey member chains.

## POST-QTLC1-LEXICAL-OWNER-AND-RESULT-TYPING-012

Status: implemented; qtlc1 self-check advanced to the next blocker.

- Infer expression-bodied callable results for `This`, constructor literals,
  primitive literals, and homogeneous `match` results.
- Preserve declared parameter types in callable scopes.
- Resolve constructor-initialized local types through the symbol table.
- Keep expression-bodied callable parameters and owner members in scope.
- Reuse the active lexical stack slot for sibling blocks. Appending a new slot
  for every sibling block restored a stale parent and dropped locals declared
  before the previous block.
- Keep qtlc1 semantics intact. No source rewrite or permissive member fallback
  was added to bypass a qtlc0 limitation.

Verified:

```text
qtlc/build/compiler/driver/qtlc check qtlang/compiler
qtlc/build/compiler/driver/qtlc build qtlang/compiler
```

Both commands pass for all 625 compiler files, and qtlc1 builds successfully.
The qtlc1 self-check no longer reports `products.statusText`,
`nativePlan.unitCount`, or `io.buildNativeCompiler`.

Current self-check blocker:

```text
unresolved member 'rows.start'
```

The next implementation item is generic field receiver/member resolution for
collection views such as `Array<T, N>.start => rows.start`, where `rows` is
`Range<T>`. Fix the semantic generic-owner/member path; do not expose private
storage, flatten the collection model, or weaken qtlc1 source.

## POST-QTLC1-STRUCTURED-DECL-PARAMETERS-012B

Status: next. This replaces temporary AST parameter recovery with a real
declaration-table contract.

Problem:

- Expression-bodied functions such as
  `SelfHostLinkProductPlan.fromNative(nativePlan: SelfHostNativeProductPlan) =`
  must bind `nativePlan` from the canonical declaration record, not by scanning
  nearby AST rows during scope construction.
- The current `TypeScopeTable.addAstParameterBindings` path is recovery logic.
  It is useful for proving the missing data path, but it is not the final qtlc1
  design and must not remain in the hot path.

Implementation command:

```text
POST-QTLC1-STRUCTURED-DECL-PARAMETERS-012B
  goal:
    make DeclarationRecord.parameterRows authoritative for all callable forms,
    including expression-bodied functions, block-bodied functions, impl
    methods, impl view methods, constructors/factories, and generic callables.

  parser/declaration work:
    audit SourceParser callable declaration emission
    ensure AstCallableParameter rows are emitted before both '=' bodies and
    '{' bodies
    ensure DeclarationParserBridge copies every AstCallableParameter row into
    DeclarationRecord.parameterRows without dropping expression-bodied forms
    preserve parameter name, declared type, default expression, variadic flag,
    source span, and ordinal
    preserve generic parameter rows for the same callable forms

  symbol work:
    TypeSymbol.fromRecord must derive parameters and parameterRows only from
    DeclarationRecord rows
    callableParameterCount must use structured rows, not legacy name vectors
    symbol indexes must key arity from structured parameters

  scope work:
    TypeScopeTable.addParameterBindings must bind only TypeSymbol.parameterRows
    remove addAstParameterBindings from normal scope construction
    remove astParameterTypeNameAt from the hot path
    remove any "if structured missing, scan AST" branch from function scope
    creation
    if structured parameter rows are missing, emit a deterministic frontend or
    declaration diagnostic; do not silently recover

  tests:
    add expression-bodied method parameter proof:
      pub fromNative(nativePlan: SelfHostNativeProductPlan) =
        This { objectUnitCount: nativePlan.unitCount }
    add expression-bodied generic method parameter proof
    add impl view expression-bodied parameter proof
    add negative proof that missing structured parameter rows fail the
    declaration contract before typechecking

  proof commands:
    qtlc/build/compiler/driver/qtlc check qtlang/compiler
    qtlc/build/compiler/driver/qtlc build qtlang/compiler
    qtlang/compiler/build/debug/qtlc1 check qtlang/compiler
    qtlang/compiler/build/debug/qtlc1 build qtlang/compiler

  success:
    qtlc1 self-check resolves nativePlan.unitCount through structured
    parameter binding
    no addAstParameterBindings recovery remains in TypeScopeTable hot path
    no permissive member fallback is added
    qtlc1 semantics are stricter, not weaker
```
