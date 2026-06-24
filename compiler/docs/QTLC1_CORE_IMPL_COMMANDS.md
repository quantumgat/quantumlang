# qtlc1 Core Data Implementation Commands

Status: implementation command list for replacing scalar-only compiler state
with compiler-owned data structures without depending on runtime/stdlib

Goal: qtlc1 must use real compiler data concepts such as arenas, ranges,
slices, indexes, maps, and interned names, but it must not duplicate final
stdlib collection work or depend on old stdlib/runtime packages.

Related core-language contracts:

```text
QTLC1_CORE_BYTES_DESIGN.md
QTLC1_CORE_TEXT_DESIGN.md
QTLC_BYTES_HASH_STAGE_ROADMAP.md
```

## Rule

qtlc1 owns compiler data contracts.

qtlc1 does not own final collection implementations.

This means qtlc1 source may use compiler-facing names:

```text
Arena<T>
Vec<T>
Slice<T>
Range<T>
Map<K,V>
Index<K,V>
NameId
StableId
```

`Option<T>` and `Result<T,E>` are not redefined by qtlc1 `core`; they are
language/core carrier types already supported by qtlc0. qtlc1 should use them
directly for optional and fallible compiler state.

The qtlc1-owned data structures must lower to qtlc0-safe handles, ranges, rows,
and indexes until qtlc2/qtlc3 can provide the final runtime/stdlib
implementations.

## No Double Work Contract

Do not import old stdlib collections into qtlc1:

```text
qtlc/qtlang/stdlib/collections/vec
qtlc/qtlang/stdlib/collections/map
qtlc/qtlang/stdlib/memory
qtlc/qtlang/runtime
```

Do not create temporary APIs that must be renamed later.

Use final compiler-facing names now, with bootstrap-safe internals:

```text
good: DeclarationArena, SymbolArena, AstArena, NameId, Index<K,V>
bad: DeclarationCountBag, qtlc1DeclarationRows2, temporaryVecLikeThing
```

Do not pass large nested aggregates by value through qtlc0 native ABI. The
active frontend product already proved this can crash when nested records become
too large. Prefer handles, IDs, ranges, and indexes.

## Constructor Literal Style

Multiline object/constructor literals are newline separated:

```qn
This {
  index: Index<K, V>(keys, values)
  entryCount: keys.count
}
```

Inline object/constructor literals are comma separated:

```qn
This { index: Index<K, V>(keys, values), entryCount: keys.count }
```

This keeps normal qtlc1 source clean while preserving unambiguous inline
syntax. Core constructors must lower to direct native aggregate operations,
not runtime map/reflection calls.

## Target Directory

```text
qtlang/compiler/core/
  mod.qn
  id.qn
  range.qn
  slice.qn
  vec.qn
  arena.qn
  index.qn
  map.qn
  interner.qn
  qualifiedName.qn
```

This module is compiler-local. It is not `std::collections`.

## Correct Redesign Foundation

Before fixing one-by-one compiler directories, define the shared compiler
foundation:

```text
qtlang/compiler/core/
  typed IDs
  NameId / StringInterner
  QualifiedName
  Range<T>
  Slice<T>
  Arena<T>
  Vec<T>
  Map<K,V>
  Index<K,V>
```

Use language/core `Option<T>` and `Result<T,E>` for carrier state. Do not add
compiler-local `Option` or `Result` structs to this module.

## Core Literal And Prelude Rules

qtlc1 must not require users or compiler source files to import basic language
literal capability. These are core/prelude features, not normal package APIs:

```text
vec<T>
Array<T,N>
Map<K,V>
anonymous records
target-typed object literals
Option<T>
Result<T,E>
none/some
ok/error
primitive types
```

Literal typing rules required before deep `core` migration:

```text
let values = [1, 2, 3]
  -> vec<i64> by default

let values: vec<i64> = [1, 2, 3]
  -> vec<i64>

let values: Array<i64, 3> = [1, 2, 3]
  -> fixed Array<i64,3>

let values: Array = [1, 2, 3]
  -> shorthand allowed later; compiler infers Array<i64,3>

let table = {
  "id": { id: "u1" }
}
  -> Map<string, anonymous-record>

let table: Map<string, User> = {
  "id": { id: "u1" }
}
  -> Map<string, User>; nested record is target-typed as User

let user = {
  id: "u1"
  name: "Ali"
}
  -> anonymous record/object

let user: User = {
  id: "u1"
  name: "Ali"
}
  -> User object literal
```

Disambiguation rule:

```text
[...]                  is vec<T> unless target type is Array<T,N>
target Array<T,N>      makes [...] a fixed array
{ identifier: value }  is anonymous record/object
{ "key": value }       is Map<string,T>
target User + { ... }  is User object
target Map<K,V> + { ... } is Map<K,V>
```

This matters for `core`: `Arena<T>`, `Vec<T>`, `Map<K,V>`, `Index<K,V>`,
and compiler facts should be written in final Quantum style. If qtlc0 cannot
type or lower one of these literal forms, fix qtlc0 instead of weakening qtlc1
source into count/string-only code.

Then migrate qtlc1 directories in this order:

```text
1. core IDs + Range + NameId
2. source spans/file IDs
3. diagnostics arena
4. syntax AST arena
5. frontend declaration arena
6. module graph/index
7. symbol/type arenas
8. semantic model
9. HIR/MIR arenas
10. query products bound to arena handles
```

## Existing Coverage And Missing Gaps

This roadmap must not duplicate work that already exists in qtlc0 or in the
current qtlc1 scaffold.

Already available in qtlc0:

```text
AstArena, HirArena, MirArena, TirArena exist as C++ compiler internals
Vec<T>, Map<K,V>, Slice<T>, Array<T> are recognized by type/backend paths
Option<T> and Result<T,E> have backend carrier support
collection literals, carrier constructors, and indexing have qtlc0 fixtures
```

Already available in qtlc1:

```text
StableId and Stable*Id wrappers under query/
TypeId under typeSystem/types/
AstNodeId under syntax/ast/
ModuleGraph, DiagnosticBag, DeclarationTable, TypeSymbolTable scaffolds
literal/type facts for Vec, Map, Range, Option, and Result
```

Implemented in active qtlc1 compiler state:

```text
compiler/core module
central compiler-local typed ID model
generic Range<T> and Slice<T> contracts
compiler-local Arena<T>, Vec<T>, Map<K,V>, Index<K,V> contracts
DeclarationArena and DeclarationIndex
SourceMap = Arena<SourceFile> + Index<NameId, FileId>
SourceFileTable typed file/path/module-path row handles
DeclarationSet counts as Slice<DeclarationId> views
DeclarationTable parser rows wired into DeclarationArena/DeclarationIndex
DeclarationRecordTable owns record/ID slices instead of scalar count mirrors
DeclarationIndex carries Index<DeclarationKey, DeclarationId> as final lookup spine
NameId, QualifiedName, and StringInterner
```

Missing in active qtlc1 compiler state:

```text
AstArena and AstRoot ranges
SymbolArena, TypeArena, SymbolIndex, TypeIndex
real ModuleArena and indexed ModuleGraph
DiagnosticArena and DiagnosticBag ranges
query products bound to arena/index handles
SourceFileTable still carries temporary host string row payloads until qtlc1
  owns runtime directory scanning and source reading directly
DeclarationParserRows still bridges parser summaries until parser emits complete
  concrete DeclarationRecord rows and populates the DeclarationIndex lookup
```

Important: the missing items are not new public stdlib features. They are the
compiler-facing contracts needed to make qtlc1 stop using raw counts, raw
strings, and ad hoc sentinel status values as active compiler state.

## Per-Module Data Migration Matrix

This list is the source of truth for where qtlc1 must use `Arena`, `Vec`,
`Map`, `Index`, `Range`, `Slice`, `Option`, `Result`, and typed errors. Do not
replace these with count-only records unless the file is explicitly a report or
proof bridge.

```text
source/
  use:
    FileId
    PathId or NameId for interned paths
    SourceSpan = FileId + Range<u8>
    LineMap = Vec<usize> or Array<usize> for line starts
    SourceMap = Arena<SourceFile> + Index<PathId, FileId>
  carriers:
    read(path) -> Result<SourceReadProduct, SourceError>
    lookup(path) -> Option<FileId>
  replace:
    raw fileId/start/end triples
    fileCount-only source summaries
    empty-string path sentinels

diagnostics/
  use:
    DiagnosticId
    DiagnosticArena = Arena<Diagnostic>
    DiagnosticBag = Slice<DiagnosticId> or Range<DiagnosticId>
    DiagnosticLabel = SourceSpan + message
    DiagnosticIndex = Index<FileId, Slice<DiagnosticId>>
  carriers:
    render/write -> Result<DiagnosticOutput, DiagnosticError>
    firstError -> Option<DiagnosticId>
  replace:
    diagnostic count as primary state
    raw file/start/end fields in labels

syntax/
  use:
    TokenId, TriviaId, AstNodeId
    TokenStream = Vec<Token>
    TriviaStream = Vec<Trivia>
    AstArena = Arena<AstNode>
    AstChildren = Slice<AstNodeId>
    token/trivia/node spans = SourceSpan
  carriers:
    lex(file) -> Result<TokenStreamId, LexError>
    parse(tokens) -> Result<AstRoot, ParseError>
    lookahead -> Option<TokenId>
  replace:
    token/node count-only products
    parser text-count shortcuts
    raw offset fields outside SourceSpan

frontend/
  use:
    DeclarationId
    DeclarationArena = Arena<DeclarationRecord>
    DeclarationIndex = Index<DeclarationKey, Slice<DeclarationId>>
    DeclarationTable = arena + index + module slices
    DeclarationSet = view over Slice<DeclarationId>, not stored counts
  carriers:
    declarations(ast) -> Result<DeclarationTable, FrontendError>
    find(name) -> Option<DeclarationId>
  replace:
    DeclarationSet as count bag
    DeclarationParserRows in active frontend products
    large nested declaration samples returned by value

moduleSystem/
  use:
    ModuleId, PackageId, ExportSurfaceId, ImportEdgeId
    ModuleArena = Arena<ModuleNode>
    ModuleGraph = Vec<ModuleEdge> + Index<QualifiedName, ModuleId>
    ExportSurface = Slice<SymbolId>
    ImportIndex = Index<ImportPath, ExportSurfaceId>
  carriers:
    buildGraph(package) -> Result<ModuleGraph, ModuleError>
    resolveModule(path) -> Option<ModuleId>
  replace:
    module count summaries
    string-only import/export identity

nameResolve/
  use:
    NameId
    QualifiedName = Slice<NameId> + local NameId
    SymbolIndex = Index<QualifiedName, SymbolId>
    ScopeArena = Arena<Scope>
    ScopeEntries = Map<NameId, SymbolId> or Index<NameId, SymbolId>
  carriers:
    resolve(name) -> Result<SymbolId, ResolveError>
    lookupLocal(name) -> Option<SymbolId>
  replace:
    repeated hot-path strings
    unknown-name sentinels

typeSystem/
  use:
    TypeId, SymbolId, TypeFactId, TypeContextId
    TypeArena = Arena<CompilerType>
    TypeFactArena = Arena<TypeFact>
    TypeContextArena = Arena<TypeContext>
    TypeIndex = Index<QualifiedName, TypeId>
    LayoutIndex = Index<TypeId, TypeLayoutId>
    TypeEnvironment = arenas + indexes + slices
  carriers:
    resolveType(name) -> Result<TypeId, TypeError>
    inferExpr(expr) -> Result<TypeId, TypeError>
    findType(name) -> Option<TypeId>
  replace:
    TypeEnvironment count-derived state
    string status fields for type success/failure
    duplicate TypeId outside core

semantic/
  use:
    CheckedFileId, CheckedModuleId, PublicApiId
    CheckedFileArena = Arena<CheckedFile>
    CheckedModuleArena = Arena<CheckedModule>
    PublicApiArena = Arena<PublicApiSymbol>
    PublicApiIndex = Index<QualifiedName, PublicApiId>
  carriers:
    check(module) -> Result<CheckedModuleId, SemanticError>
    publicSymbol(name) -> Option<PublicApiId>
  replace:
    semantic count summaries
    public API stored as strings only

hir/
  use:
    HirItemId, HirFunctionId, HirBlockId, HirExprId, HirStmtId, HirTypeRefId
    HirArena = Arena<HirNode>
    HirModule items = Slice<HirItemId>
    HirFunction params/blocks = Slice<HirParamId>, Slice<HirBlockId>
    HirBlock statements/expressions = Slice<HirStmtId>, Slice<HirExprId>
  carriers:
    lowerAst(root) -> Result<HirModuleId, HirError>
    findFunction(name) -> Option<HirFunctionId>
  replace:
    raw i64 node IDs
    string-only HIR kind/state fields

mir/
  use:
    MirBodyId, MirBlockId, MirInstructionId, MirValueId
    MirBodyArena = Arena<MirBody>
    MirBlockArena = Arena<MirBlock>
    MirInstructionArena = Arena<MirInstruction>
    MirValueArena = Arena<MirValue>
    MirBody blocks/values/instructions = Slice<...Id>
  carriers:
    lowerHir(function) -> Result<MirBodyId, MirError>
    terminator(block) -> Option<MirTerminatorId>
  replace:
    block/instruction/value count-only state
    raw i64 IDs in hot MIR data

controlFlow/
  use:
    ControlFlowRegionId, LoopFactId, FlowEdgeId
    RegionArena = Arena<ControlFlowRegion>
    LoopFactArena = Arena<LoopFact>
    FlowEdges = Vec<ControlFlowEdge>
    BlockToRegion = Index<MirBlockId, ControlFlowRegionId>
  carriers:
    analyze(mir) -> Result<ControlFlowGraph, ControlFlowError>
    loopFor(block) -> Option<LoopFactId>
  replace:
    raw region/loop IDs
    string loop/control status values

query/
  use:
    QueryKey, ProductId, DependencyEdgeId, DirtyReasonId
    QueryIndex = Index<QueryKey, ProductId>
    ProductArena = Arena<QueryProduct>
    DependencyGraph = Vec<DependencyEdge>
    DirtySet = Index<QueryKey, DirtyReasonId>
  carriers:
    execute(key) -> Result<ProductId, QueryError>
    cached(key) -> Option<ProductId>
  replace:
    product identity as string path only
    all-or-nothing package invalidation

buildSystem/
  use:
    BuildTargetId, BuildStepId, ArtifactId
    BuildPlan = Arena<BuildStep> + Slice<BuildStepId>
    ArtifactIndex = Index<ArtifactKey, ArtifactId>
    LinkInputSet = Slice<ObjectUnitId> or Vec<ObjectUnitId>
  carriers:
    plan(target) -> Result<BuildPlanId, BuildError>
    artifact(key) -> Option<ArtifactId>
  replace:
    build phase status strings
    link input counts without object IDs

backend/native/
  use:
    NativeObjectUnitId, NativeObjectUnitKey, SectionId, RelocationId
    ObjectUnitArena = Arena<NativeObjectUnit>
    ObjectUnitIndex = Index<NativeObjectUnitKey, NativeObjectUnitId>
    ObjectSections = Slice<SectionId>
    ObjectBytes = Range<u8>
    LinkInputs = Slice<ObjectUnitId>
  carriers:
    emitObject(mir) -> Result<ObjectProduct, BackendError>
    link(inputs) -> Result<LinkProduct, LinkError>
    objectFor(key) -> Option<NativeObjectUnitId>
  replace:
    string cache keys as primary identity
    object/link count summaries

backend/quantum/
  use:
    QuantumCircuitId, QuantumOperationId, QubitId, QregId, QuantumGateId
    QuantumOperations = Vec<QuantumOperation>
    QuantumTargets = Slice<QubitId> or Slice<QregId>
    GateIndex = Index<NameId, QuantumGateId>
    HardwareLayout = Array<QubitId> only when target size is fixed
  carriers:
    verify(circuit) -> Result<QuantumCircuitId, QuantumError>
    gate(name) -> Option<QuantumGateId>
  replace:
    quantum resource/effect counts as active state
    target topology strings in hot path

interface/
  use:
    QiDeclarationId, QcapSectionId, QidxEntryId
    QiFile declarations = Slice<QiDeclarationId>
    QcapProduct sections = Slice<QcapSectionId>
    Qcap section bytes = Range<u8>
    QidxProduct = Index<QualifiedName, QidxEntryId>
  carriers:
    writeQi(file) -> Result<QiProduct, InterfaceError>
    readQcap(path) -> Result<QcapProduct, InterfaceError>
    findExport(name) -> Option<QidxEntryId>
  replace:
    interface count summaries
    `.qnapi` legacy assumptions

machine/
  use:
    RegisterClassId, CallingConventionId, AbiSlotId
    fixed ABI registers = Array<RegisterClass>
    target feature lookup = Index<NameId, TargetFeatureId>
    ABI slot list = Slice<AbiSlotId>
  carriers:
    lowerAbi(type) -> Result<AbiLayout, MachineError>
    feature(name) -> Option<TargetFeatureId>
  replace:
    free-form target feature strings
    ABI field counts without typed slots

platform/
  use:
    PlatformId, MemoryRegionId, TargetFeatureId
    EmbeddedMemoryMap = Vec<MemoryRegion>
    MemoryRegion bytes = Range<u8>
    PlatformFeatureIndex = Index<NameId, TargetFeatureId>
  carriers:
    detect(target) -> Result<PlatformFact, PlatformError>
    memoryRegion(name) -> Option<MemoryRegionId>
  replace:
    platform strings as identity
    embedded region counts without ranges

driver/
  use:
    Command, Session, ExitCode as small value types
    target argument = Option<string> when absent is meaningful
    command table = Map<NameId, Command> or Index<NameId, CommandId> later
  carriers:
    run(args) -> Result<ExitCode, DriverError>
    targetPath -> Option<string>
  replace:
    provided bool + empty target string when qtlc0 supports the cleaner carrier path

tests/
  use:
    focused proof packages for each data contract
    active compiler tests must exercise Arena/Vec/Map/Index/Option/Result
  carriers:
    proof helpers may summarize counts, but real compiler products may not
  replace:
    proof-only text/count shortcuts before self-host qtlc2 proof
```

## Command List

### POST-QTLC1-CORE-LITERAL-PRELUDE-001

Make core literal and carrier capability available without imports.

Status:

```text
implemented:
  qtlc0 proof app covers vec, fixed Array, Map literal syntax, target-typed
  object literals, Option<T>, and Result<T,E>
  qtlc0 parser accepts newline-separated object/map fields
  qtlc0 type checker infers quoted-key map literals and anonymous records
  qtlc0 native lowering accepts lowercase vec<T> and materializes Map handles
  qtlc0 runtime ABI provides QnMapSet/QnMapInsert/QnMapGetU64 for real map entries
  qtlc0 native map literals insert quoted-key entries through QnMapSet
  qtlc0 native backend exports synthetic anonymous-record aggregate layouts
  qtlc0 per-unit native object restore maps compact body units to .native.o
  qtlc0 post-compiler-change proof restored 14 objects and compiled 0
  qtlc1 post-qtlc0-rebuild proof attached 318 fallback object keys and
  restored 318 native objects with compiled_objects=0
  qtlc0 frontend-state execution now accepts source-stable full hydrate after
  compiler-identity changes; checked-source builder skips parse=0/check=0 and
  hydrates live CheckedSourceSet state from validated frontend artifacts
  qtlc0_test_build proof: hydrate=15 parse=0 check=0, builder skipped, backend
  product restored, fresh_objects=1, restored_objects=14, compiled_objects=0
  qtlc1 proof: hydrate=422 parse=0 check=0, builder skipped, 413 lowered units
  restored, 9 lowered units rebuilt, fresh_objects=104, restored_objects=318,
  compiled_objects=0
  qtlc1 core uses Array<T,N>
  qtlc1 core proof covers Vec/Array/Map/Slice plus Option/Result carriers

remaining:
  qtlc1 source modules still need migration from count-only facts to real
  Vec/Map/Arena/Range/Slice/Option/Result products
  partial changed-source frontend merge still needs conservative recheck until
  qtlc1 declaration/type dependencies are precise
  backend emit still walks the assembled lowered product; next speed work must
  restore emit products from unchanged lowered-unit shards
```

Details:

```text
vec<T>, Array<T,N>, Map<K,V>, Option<T>, and Result<T,E> are core/prelude names
anonymous record literals require no import
target-typed object literals require no import
none/some and ok/error carriers require no import
primitive types require no import
print family remains prelude where already intended
qtlc1 source must not import stdlib/runtime for these features
```

Required literal typing:

```text
let values = [1, 2, 3]
  -> vec<i64>

let values: vec<i64> = [1, 2, 3]
  -> vec<i64>

let values: Array<i64, 3> = [1, 2, 3]
  -> fixed Array<i64,3>

let table = {
  "id": { id: "u1" }
}
  -> Map<string, anonymous-record>

let table: Map<string, User> = {
  "id": { id: "u1" }
}
  -> Map<string, User>

let user = {
  id: "u1"
  name: "Ali"
}
  -> anonymous record/object

let user: User = {
  id: "u1"
  name: "Ali"
}
  -> User object literal
```

Implementation checks:

```text
if qtlc0 cannot parse/check/lower these forms, fix qtlc0
do not weaken qtlc1 source to helper constructors or count-only records
add qtlc0_test_build proof for vec/Array/Map/anonymous-record/object literal
add qtlc1 compiler proof that core can use these literals directly
prove no runtime/stdlib import is needed
```

### POST-QTLC0-AUTO-CONSTRUCTOR-DEFAULT-CONTRACT-001

Lock structural auto-constructor semantics for qtlc1 and user code.

Details:

```text
pub Type { fields... } creates an auto-constructor without requiring impl Type
required fields are positional/named required parameters
defaulted fields are optional positional/named parameters
positional arguments must come before named arguments
Type(...) supports default filling with no helper constructor
Type { ... } supports positional plus named fields with default filling
let value: Type = { ... } supports target-typed positional plus named fields
multiline object literals may use newline separators; commas are optional
inline object literals must use commas between fields
all-default types support Type()
manual empty constructors like Type() = This() are recursive and must be rejected
custom constructors remain allowed for validation/computed defaults
generated native code must be direct aggregate stores with no reflection,
named-argument map, runtime constructor table, or dynamic dispatch
proof: qtlang/qtlc0_test_build/src/autoConstructorDefaultProof.qn
```

### POST-QTLC1-CORE-DATA-CONTRACT-001

Create `compiler/core` module and define the ownership boundary.

Details:

```text
add core/module.toml and mod.qn
add contract files for CoreKind, coverage, gaps, migration, and boundary
no dependency on runtime or stdlib collections
all exports use Quantum module export style
record existing qtlc0 support vs missing qtlc1 data contracts
qtlc0 check qtlang/compiler/core
```

### POST-QTLC1-CORE-DATA-ID-RANGE-002

Add stable compiler IDs and range facts.

Details:

```text
define PackageId, ModuleId, FileId, NodeId, DeclarationId, SymbolId, TypeId
centralize current scattered Stable*Id, TypeId, AstNodeId, and raw u64/i64 IDs
define NameId as interned-name handle
define Range<T> with start/count/end/empty/contains views
add CoreAudit area records so all count/raw-id/string-status migrations stay visible
avoid strings as identity in hot compiler state
```

Initial audit areas that must migrate after this command:

```text
source:
  SourceSpan, LineMap, SourceMap, SourceReader use raw fileId/fileCount

diagnostics:
  DiagnosticBag and TypeDiagnosticSet are count-only
  DiagnosticLabel uses raw fileId/start/end

syntax:
  AstNodeId, Token, Trivia, Scanner, Lexer, Parser use raw fileId/count fields
  AstNode product state is not arena-backed yet

frontend:
  DeclarationSet, DeclarationTable, DeclarationRecordRange, FrontendProduct
  still use counts and synthetic ranges

moduleSystem:
  ModuleGraph, PackageGraph, ExportSurface are count summaries

typeSystem:
  TypeSymbolTable, TypeEnvironment, TypeRegistry, TypecheckState are count-derived
  TypeId exists separately and must migrate to core TypeId

semantic:
  SemanticModel, CheckedFile, CheckedModule, PublicApi are count summaries

hir/mir/controlFlow:
  HIR/MIR nodes, blocks, values, loops, regions use raw i64 IDs and string kinds

query/build/backend:
  query products still store counts and StableId wrappers outside core
  native/link products still use count summaries and string cache keys

quantum:
  QIR/QMIR/resource/control facts still use node/resource/effect counts
```

Whole-tree audit snapshot after `POST-QTLC1-CORE-DATA-ID-RANGE-002`:

```text
count fields by area:
  query       85
  frontend    59
  typeSystem  50
  interface   43
  buildSystem 36
  quantum     35
  moduleSystem 26
  backend     22
  semantic    20
  hir         17
  mir         15
  syntax      14

raw ID fields by area:
  syntax      43
  hir         38
  controlFlow 37
  mir         14
  frontend     8
  source       7

string kind/status fields by area:
  typeSystem 36
  hir        23
  buildSystem 23
  mir        13
  quantum     8

span/offset fields by area:
  syntax      27
  source       8
  frontend     6
  diagnostics  5
```

Audit rule:

```text
count-only records are allowed in reports, not active compiler state
raw IDs must become typed IDs before arena migration
file/start/end groups must become SourceSpan or Range<byte>
string kind/status values must become enums or Option/Result carriers
large row sets must become Range<T>, Slice<T>, Vec<T>, Arena<T>, Map<K,V>, or Index<K,V>
```

### POST-QTLC1-CORE-DATA-SLICE-VEC-SHELL-003

Add compiler-local `Slice<T>`, `Vec<T>`, and `Array<T>` contracts.

Details:

```text
Slice<T> is a borrowed view over arena/vector rows
Vec<T> is an ordered compiler row contract
Array<T> is a fixed-size compiler row contract
no heap allocator implementation yet
qtlc0-safe lowering uses row counts, ranges, and handles
add slice.qn, vec.qn, and array.qn
export Slice<T>, Vec<T>, and Array<T> from core
record that this is a shell contract, not std::collections
```

### POST-QTLC1-CORE-DATA-ARENA-SHELL-004

Add compiler-local `Arena<T>` contract.

Details:

```text
Arena<T> stores compiler records by typed ID
records are addressed by DeclarationId, NodeId, SymbolId, TypeId, etc.
no large nested product values
no pass-by-value record arrays through qtlc0 ABI
add arena.qn
Arena<T> uses Range<T>, capacity, owner CompilerId, and generation metadata
```

### POST-QTLC1-CORE-DATA-INDEX-MAP-SHELL-005

Add compiler-local `Index<K,V>` and `Map<K,V>` contracts.

Details:

```text
Index<K,V> is deterministic compiler lookup state
Map<K,V> is semantic source-level map contract
keep hashing/storage abstract in qtlc1
support lookup facts needed by modules, symbols, declarations, and queries
add index.qn and map.qn
Map<K,V> is backed by Index<K,V> shell metadata, not runtime hash storage
```

### POST-QTLC1-CORE-DATA-CARRIER-USE-006

Use language/core `Option<T>` and `Result<T,E>` contracts.

Details:

```text
Option<T> models optional compiler values without sentinel strings
Result<T,E> models fallible compiler products without ad hoc status strings
none/some and ok/error facts must be type-checkable by qtlc1
do not define compiler-local Option<T> or Result<T,E> structs
reuse the qtlc0 language/backend carrier support
no large nested Result payloads in active qtlc0 ABI paths
start replacing string status values in StageStatus and TypecheckState later
```

### POST-QTLC1-CORE-DATA-INTERNER-007

Add name interning model.

Details:

```text
NameId replaces repeated hot-path strings
QualifiedName stores module path plus local name IDs
StringInterner model records deterministic fingerprints
no runtime string table implementation yet
add qualifiedName.qn and interner.qn
StringInterner uses Vec<NameId>, Vec<u8>, and Index<NameId, Range<u8>>
```

### POST-QTLC1-SOURCE-DATA-CONTRACT-001

Move source identity to typed IDs and ranges.

Status:

```text
started:
  SourceFile now uses FileId.
  SourceSpan now stores FileId + Range<u8>.
  LineMap now carries Vec<u64> line-start state instead of only a scalar count.
  SourceMap now carries Arena<SourceFile> + Index<NameId, FileId>.
  source module check passes.
  full qtlc check qtlang/compiler passes with 425 files after the typed source migration.
  frontend DeclarationRecord now carries FileId + Range<u8> instead of raw fileId/start/end
  so source coordinates no longer leak into active frontend records as scalar triples.

remaining:
  intern path/module identity away from raw string paths into NameId/QualifiedName use
  replace source summary-only products with SourceMap-backed package state
  move source lookup APIs to Option<FileId>
  move source read/extract APIs to Result<..., SourceError>
  replace remaining frontend/syntax raw source coordinate mirrors with SourceSpan-backed state
```

Details:

```text
SourceSpan becomes FileId + Range<u8>
SourceFile uses FileId plus interned PathId/NameId identity
LineMap uses Vec<usize> or Array<usize> for line starts
SourceMap uses Arena<SourceFile> + Index<PathId, FileId>
source reads return Result<SourceReadProduct, SourceError>
source lookups return Option<FileId>
```

### POST-QTLC1-DIAGNOSTIC-DATA-CONTRACT-001

Move diagnostics to arena-backed state.

Details:

```text
DiagnosticArena stores Diagnostic by DiagnosticId
DiagnosticBag stores Slice<DiagnosticId> or Range<DiagnosticId>
DiagnosticLabel uses SourceSpan
DiagnosticIndex maps FileId to diagnostic ranges
render/write APIs return Result<..., DiagnosticError>
first error lookup returns Option<DiagnosticId>
```

### POST-QTLC1-SYNTAX-DATA-CONTRACT-001

Move lexer/parser products to token streams and AST arenas.

Details:

```text
TokenStream uses Vec<Token>
TriviaStream uses Vec<Trivia>
AstArena stores AstNode by AstNodeId
AstChildren uses Slice<AstNodeId>
token/trivia/node location uses SourceSpan
lex and parse return Result products
lookahead returns Option<TokenId>
```

### POST-QTLC1-FRONTEND-DATA-CONTRACT-001

Move declaration products to arena and index state.

Status:

```text
started:
  DeclarationSet now carries Slice<DeclarationId> ranges for all declarations,
  public declarations, and unresolved declarations while retaining scalar mirror
  counts for current qtlc0 view-hydration compatibility.
  DeclarationArena exists and stores Arena<DeclarationRecord> plus declaration
  ID slices.
  DeclarationIndex exists and stores DeclarationKey/DeclarationId slices,
  public surface slices, and unresolved slices.
  DeclarationTable now owns DeclarationArena and DeclarationIndex.
  FrontendSourceProduct now owns PackageSourceScan plus SourceMap-backed source
  state instead of only count booleans.
  FrontendSourceProduct keeps scalar source mirrors only for current qtlc0
  imported-view compatibility while package/sourceMap remain the source of
  truth.
  FrontendDeclarationProduct keeps scalar declaration-count mirrors only for
  current qtlc0 imported-view compatibility while DeclarationTable and
  DeclarationRecordTable remain the real owned declaration state.
  frontendProductForPackage(target) now builds frontend products from
  packageSourceScanHost(target) plus RuntimeFrontendScan(target).
  qtlc check qtlang/compiler/frontend passes.
  qtlc check qtlang/compiler passes with 429 files.

bridges still present:
  packageSourceScanHost(target) exists because imported associated extract
  members like PackageSourceScan.fromHost(target) are still a qtlc0
  cross-module visibility gap.
  frontendProductForPackage(target), checkReportForTarget(target), and
  selfHostBuildQtlc2(...) exist because imported associated functions like
  FrontendProduct.forPackage(...) and SelfHostBuildReport.qtlc2(...) are still
  a qtlc0 cross-module associated-member gap.
  scalar mirror counts remain on FrontendSourceProduct and
  FrontendDeclarationProduct because imported nested impl-view properties are
  still less reliable than direct fields in some qtlc0 paths.

remaining:
  replace parser count summaries with parser-produced DeclarationRecord rows
  replace scalar mirror counts after qtlc0 recursive view hydration is complete
  move lookup from parallel qtlc0-safe slices to final Index<DeclarationKey, DeclarationId>
  add declaration lookup APIs returning Option<DeclarationId>
  add declaration extraction APIs returning Result<DeclarationTable, FrontendError>
  remove bridge helpers after qtlc0 cross-module associated-member support is fixed
```

Details:

```text
DeclarationArena stores DeclarationRecord by DeclarationId
DeclarationIndex maps DeclarationKey to Slice<DeclarationId>
DeclarationTable owns arena/index/module slices
DeclarationSet becomes an impl view over slices
declaration extraction returns Result<DeclarationTable, FrontendError>
declaration lookup returns Option<DeclarationId>
```

### POST-QTLC1-DECLARATION-ARENA-001

Replace declaration scalar rows with arena/index state.

Status:

```text
started:
  DeclarationParserRows now has fromCounts(...) so parser row ranges can be
  built directly from declaration count facts without routing through a
  count-only table first.
  DeclarationRecordRange now exposes declarationIds/publicDeclarationIds/
  unresolvedDeclarationIds so row ranges can drive DeclarationSet directly.
  DeclarationTable now builds DeclarationSet values from parser row slices
  instead of using count-only DeclarationSet(kind, count) as the primary state.
  DeclarationArena now has fromParserRows(rows, samples) and marks parser row
  plus parser sample backing explicitly.
  DeclarationIndex now has fromParserRows(rows, samples) and marks parser row
  plus parser sample backing explicitly.
  DeclarationRecordTable now carries parser-produced DeclarationParserRecordSamples
  and requires them for validity.
  DeclarationParserBridge now carries parser record samples and requires them
  for the parser-backed path to be ready.
  DeclarationRecordSource/DeclarationRecordRows now distinguish real
  parser-produced rows from runtime-bridged rows, so qtlc1 can gate later
  module/type stages on parser-owned declaration records instead of treating
  host/runtime scan rows as real parser output.
  qtlc check qtlang/compiler/frontend passes.
  qtlc check qtlang/compiler passes with 429 files.

remaining:
  replace synthetic parser row names/module paths with real parser-produced
  DeclarationRecord rows from qtlc1 frontend execution
  move DeclarationIndex from count-shaped Slice ranges toward final
  Index<DeclarationKey, DeclarationId> lookup state
  add declaration lookup APIs returning Option<DeclarationId>
  remove scalar declaration mirrors after qtlc0 cross-module view/member gaps
  are fixed
```

Details:

```text
DeclarationArena stores DeclarationRecord by DeclarationId
DeclarationIndex maps kind/module/name to declaration ranges
DeclarationParserRows becomes bridge-only or disappears
FrontendDeclarationProduct exposes structured declaration state
```

### POST-QTLC1-AST-ARENA-001

Move AST products to arena-backed node state.

Details:

```text
AstArena stores AstNode records by NodeId
AstRoot stores root NodeId and source FileId
green/red syntax products reference NodeId ranges
parser output stops using only aggregate node counts
```

### POST-QTLC1-MODULE-DATA-CONTRACT-001

Move package/module boundary state to graph indexes.

Status:

```text
started:
  ModuleNode, ModuleEdge, ModuleArena, and ImportIndex now exist in
  moduleSystem.
  ModuleGraph owns ModuleArena, Vec<ModuleEdge>, Index<QualifiedName, ModuleId>,
  ImportIndex, and ExportSurface while keeping scalar mirrors for current
  qtlc0 compatibility.
  ModuleGraph views now validate arena/index/import backing before reporting
  incrementalReady.
  ModuleGraph now exposes lookupModule(...) -> Option<ModuleId> and
  resolveImport(...) -> Option<ModuleId> seed APIs.
  ImportIndex now has a qtlc0-safe sample lookup path and explicit lookupReady
  state.
  ExportSurface now owns DeclarationId/SymbolId slices for public API/export
  surface data instead of only scalar counts.
  ModuleGraph.fromDeclarations(table) and fromDeclarationSets(...) now build
  graph arena/index/import/export state directly from DeclarationTable /
  DeclarationSet state instead of unrelated count-only inputs.
  ModuleGraph and ImportIndex now separate row/index-backed lookup readiness
  from temporary sample/single-module fallback readiness.
  ExportSurface now reports declarationBacked and symbolBacked state over its
  DeclarationId/SymbolId slices.
  ModuleNode.fromDeclarations(...), ImportIndex.fromDeclarations(...), and
  ExportSurface.fromDeclarations(...) now consume DeclarationSet state directly.
  ModuleGraph.fromDeclarationSets(...) now uses those declaration-backed
  constructors instead of rebuilding those products from scalar counts.
  full qtlc check qtlang/compiler passes with 433 files.

remaining:
  replace count-shaped graph construction with parser-owned module records
  replace sample lookup paths with real Index-backed lookup over row storage
  add ImportIndex lookup from ImportPath to ExportSurface
```

Details:

```text
ModuleArena stores ModuleNode by ModuleId
ModuleGraph stores Vec<ModuleEdge>
module lookup uses Index<QualifiedName, ModuleId>
ExportSurface uses Slice<SymbolId>
ImportIndex maps ImportPath to ExportSurfaceId
graph construction returns Result<ModuleGraph, ModuleError>
module lookup returns Option<ModuleId>
```

### POST-QTLC1-SYMBOL-TYPE-ARENA-001

Move symbol and type state to arena/index state.

Details:

```text
SymbolArena stores Symbol records by SymbolId
TypeArena stores TypeFact records by TypeId
SymbolIndex maps QualifiedName to SymbolId
TypeIndex maps declarations and expressions to TypeId
TypeEnvironment stops being count-derived state
```

### POST-QTLC1-NAME-RESOLVE-DATA-CONTRACT-001

Move name resolution to interned names and indexes.

Status: started.

Details:

```text
QualifiedName uses Slice<NameId> plus local NameId
ScopeArena stores Scope by ScopeId with ScopeEntry rows
Scope uses Slice<ScopeEntry>, Slice<SymbolId>, Slice<NameId>, and Map<NameId, SymbolId>
SymbolIndex maps QualifiedName to SymbolId
Resolver owns ModuleGraph + ScopeArena + SymbolIndex
resolve returns Option<SymbolId> until diagnostic/name errors move to arena-backed Result
local lookup returns Option<SymbolId>
```

### POST-QTLC1-TYPESYSTEM-DATA-CONTRACT-001

Move type state to type arenas and indexes.

Details:

```text
TypeArena stores CompilerType by TypeId
TypeFactArena stores TypeFact by TypeFactId
TypeContextArena stores TypeContext by TypeContextId
TypeIndex maps QualifiedName to TypeId
LayoutIndex maps TypeId to TypeLayoutId
TypeEnvironment owns arenas/indexes and derives counts with impl view
resolve/infer return Result<TypeId, TypeError>
type lookup returns Option<TypeId>
```

### POST-QTLC1-MODULE-GRAPH-INDEX-001

Replace module boundary counts with indexed graph state.

Details:

```text
ModuleArena stores module records by ModuleId
ModuleGraph stores import/export/dependency edges
export surface uses symbol-level dependency edges
cross-module lookup goes through qcap/qidx-compatible indexes
```

### POST-QTLC1-SEMANTIC-DATA-CONTRACT-001

Move checked program state to semantic arenas.

Details:

```text
CheckedFileArena stores CheckedFile by CheckedFileId
CheckedModuleArena stores CheckedModule by CheckedModuleId
PublicApiArena stores PublicApiSymbol by PublicApiId
PublicApiIndex maps QualifiedName to PublicApiId
semantic check returns Result<CheckedModuleId, SemanticError>
public API lookup returns Option<PublicApiId>
```

### POST-QTLC1-HIR-DATA-CONTRACT-001

Move HIR bodies to typed IDs and slices.

Details:

```text
HirArena stores HIR nodes/items/functions/blocks/exprs/stmts/types
HirModule owns Slice<HirItemId> and Slice<HirFunctionId>
HirFunction owns Slice<HirParamId> and Slice<HirBlockId>
HirBlock owns Slice<HirStmtId> and Slice<HirExprId>
AST lowering returns Result<HirModuleId, HirError>
function lookup returns Option<HirFunctionId>
```

### POST-QTLC1-MIR-DATA-CONTRACT-001

Move MIR bodies to typed block/instruction/value arenas.

Details:

```text
MirBodyArena stores MirBody by MirBodyId
MirBlockArena stores MirBlock by MirBlockId
MirInstructionArena stores MirInstruction by MirInstructionId
MirValueArena stores MirValue by MirValueId
MirBody owns Slice<MirBlockId>, Slice<MirInstructionId>, and Slice<MirValueId>
HIR lowering returns Result<MirBodyId, MirError>
terminator lookup returns Option<MirTerminatorId>
```

### POST-QTLC1-CONTROL-FLOW-DATA-CONTRACT-001

Move CFG and loop state to arenas and edge vectors.

Details:

```text
RegionArena stores ControlFlowRegion by ControlFlowRegionId
LoopFactArena stores LoopFact by LoopFactId
FlowEdges use Vec<ControlFlowEdge>
BlockToRegion uses Index<MirBlockId, ControlFlowRegionId>
analysis returns Result<ControlFlowGraph, ControlFlowError>
loop lookup returns Option<LoopFactId>
```

### POST-QTLC1-DIAGNOSTIC-ARENA-001

Move diagnostics to arena-backed state.

Details:

```text
DiagnosticArena stores diagnostics by DiagnosticId
DiagnosticBag stores ranges per file/module/stage
diagnostic output references SourceSpan and stable IDs
avoid one giant diagnostic aggregate
```

### POST-QTLC1-QUERY-DATA-CONTRACT-001

Move query/cache state to typed keys, products, and dependencies.

Details:

```text
QueryIndex maps QueryKey to ProductId
ProductArena stores QueryProduct by ProductId
DependencyGraph uses Vec<DependencyEdge>
DirtySet uses Index<QueryKey, DirtyReasonId>
query execution returns Result<ProductId, QueryError>
cache lookup returns Option<ProductId>
```

### POST-QTLC1-BUILD-SYSTEM-DATA-CONTRACT-001

Move build plans and artifacts to typed step/product state.

Details:

```text
BuildPlan uses Arena<BuildStep> plus Slice<BuildStepId>
ArtifactIndex maps ArtifactKey to ArtifactId
LinkInputSet uses Slice<ObjectUnitId> or Vec<ObjectUnitId>
planning returns Result<BuildPlanId, BuildError>
artifact lookup returns Option<ArtifactId>
```

### POST-QTLC1-NATIVE-BACKEND-DATA-CONTRACT-001

Move native backend products to object-unit arenas and indexes.

Details:

```text
ObjectUnitArena stores NativeObjectUnit by NativeObjectUnitId
ObjectUnitIndex maps NativeObjectUnitKey to NativeObjectUnitId
ObjectSections use Slice<SectionId>
ObjectBytes use Range<u8>
LinkInputs use Slice<ObjectUnitId>
object emit returns Result<ObjectProduct, BackendError>
link returns Result<LinkProduct, LinkError>
object lookup returns Option<NativeObjectUnitId>
```

### POST-QTLC1-QUANTUM-BACKEND-DATA-CONTRACT-001

Move quantum state to circuit, operation, and gate indexes.

Details:

```text
QuantumOperations use Vec<QuantumOperation>
operation targets use Slice<QubitId> or Slice<QregId>
GateIndex maps NameId to QuantumGateId
fixed target layouts use Array<QubitId>
quantum verify/lower returns Result<..., QuantumError>
gate lookup returns Option<QuantumGateId>
```

### POST-QTLC1-INTERFACE-DATA-CONTRACT-001

Move `.qi`, `.qcap`, and `.qidx` products to typed sections and indexes.

Details:

```text
QiFile owns Slice<QiDeclarationId>
QcapProduct owns Slice<QcapSectionId>
Qcap section bytes use Range<u8>
QidxProduct uses Index<QualifiedName, QidxEntryId>
read/write APIs return Result<..., InterfaceError>
export lookup returns Option<QidxEntryId>
```

### POST-QTLC1-MACHINE-PLATFORM-DATA-CONTRACT-001

Move target, ABI, and platform facts to typed indexes.

Details:

```text
fixed ABI register layouts use Array<RegisterClass>
target features use Index<NameId, TargetFeatureId>
ABI slots use Slice<AbiSlotId>
embedded memory maps use Vec<MemoryRegion>
memory regions use Range<u8>
ABI lowering returns Result<AbiLayout, MachineError>
platform detection returns Result<PlatformFact, PlatformError>
feature lookup returns Option<TargetFeatureId>
```

### POST-QTLC1-DRIVER-CARRIER-DATA-CONTRACT-001

Move CLI absence/failure paths to Option and Result.

Details:

```text
target argument uses Option<string> instead of provided bool plus empty string
command table later uses Map<NameId, Command> or Index<NameId, CommandId>
Session.run returns Result<ExitCode, DriverError>
small CLI report structs may keep scalar fields
```

### POST-QTLC1-QUERY-STORE-BINDING-001

Bind core data handles to query products.

Details:

```text
query products store stable IDs, ranges, fingerprints, and product handles
no-change rebuild reuses arena/index products
private body edit invalidates only affected ranges/products
public symbol edit invalidates exact exported-symbol dependents
```

### POST-QTLC1-CORE-DATA-QTLC0-ABI-GUARD-001

Add proof that core data products stay qtlc0 ABI-safe.

Details:

```text
no large nested aggregate pass/return in active frontend products
hot products use handles/ranges/IDs
native build and qtlc1 check must not segfault
keep qtlc1 source strong; fix qtlc0 when a real language feature is missing
```

### POST-QTLC1-CORE-DATA-PROOF-001

Add focused proof package/source for compiler data contracts.

Details:

```text
prove Arena<T>, Vec<T>, Slice<T>, Range<T>, Map<K,V>, Index<K,V> parse/check
prove language Option<T> and Result<T,E> parse/check without ad hoc string status
prove view properties work across module import/export
prove constructors/default fields/copy-update stay valid
prove direct qtlc0 check/build still succeeds
```

### POST-QTLC1-DECLARATION-INCREMENTAL-PROOF-001

Prove declaration arena/index supports precise invalidation.

Details:

```text
one private declaration body edit keeps public declaration index stable
one public declaration name/signature edit changes exact symbol identity
qtlc1 check reports changed declaration/product IDs
unchanged declaration records replay from cache
```

## Migration Order

Use this order to avoid double work:

```text
1. core contracts
   IDs, NameId, Range<T>, Slice<T>, Vec<T>, Arena<T>, Map<K,V>, Index<K,V>,
   QualifiedName, StringInterner; use language Option<T> and Result<T,E>

2. source identity
   SourceFile uses FileId
   SourceSpan uses FileId + Range<byte>
   LineMap and SourceMap stop exposing raw offsets as primary state

3. diagnostics
   DiagnosticArena stores diagnostics by DiagnosticId
   DiagnosticBag stores Range<DiagnosticId>
   DiagnosticLabel references SourceSpan

4. syntax
   token/trivia streams use Vec<Token> and Vec<Trivia>
   AstArena stores AstNode by NodeId
   AstRoot stores root NodeId and child Range<NodeId>

5. frontend declarations
   DeclarationArena stores DeclarationRecord by DeclarationId
   DeclarationIndex maps module/name/kind to Range<DeclarationId>
   DeclarationSet becomes a view over ranges, not count storage

6. module graph
   ModuleArena stores modules by ModuleId
   ModuleGraph uses Index<ModuleId, ImportEdge> and ExportSurface indexes

7. symbol and type state
   SymbolArena, TypeArena, SymbolIndex, TypeIndex
   TypeEnvironment stops being aggregate-count state

8. semantic model
   CheckedFile, CheckedModule, PublicApi reference typed ranges and indexes

9. HIR/MIR/controlFlow
   HIR/MIR nodes, bodies, blocks, values, loops, regions use typed IDs
   body/block contents use ranges into arenas

10. query/build/backend
   query products store typed IDs, ranges, fingerprints, and product handles
   native/backend products use object-unit IDs and ranges, not string keys only

11. qtlc1 self-host proof
   qtlc1 builds qtlc2 with arena/index products
```

Do not rewrite everything at once. Each command must leave:

```text
qtlc/build/compiler/driver/qtlc check -j8 qtlang/compiler
qtlc/build/compiler/driver/qtlc build -j8 qtlang/compiler
qtlang/compiler/build/debug/qtlc1 check qtlang/compiler
```

green or explicitly document the blocker.

## Success Criteria

This roadmap is complete when qtlc1 no longer depends on scalar-only compiler
state for real frontend/type/module products.

Expected final shape:

```text
DeclarationTable -> DeclarationArena + DeclarationIndex
TypeSymbolTable  -> SymbolArena + TypeArena + SymbolIndex
Module facts     -> ModuleArena + ModuleGraph
AST facts        -> AstArena + AstRoot ranges
Diagnostics      -> DiagnosticArena + DiagnosticBag
Status carriers  -> language Option<T> + Result<T,E>, not sentinel strings
```

The source API should already look like the final compiler design, while the
storage implementation remains bootstrap-safe until qtlc2/qtlc3 replace it with
faster runtime-backed storage.

## Hash Foundation

The complete staged command split is locked in
[QTLC_BYTES_HASH_STAGE_ROADMAP.md](QTLC_BYTES_HASH_STAGE_ROADMAP.md).

```text
POST-QTLC1-CORE-HASH-FOUNDATION-001
  canonical core/hash module
  digest width, algorithm, domain, policy, state, and hash-table contracts
  keep Hash128/Hash256/Hash512 as binary digest carriers
  use Blake3-256 for compiler content/action defaults
  use explicit domain separation for cache, ABI, package, and security hashes

POST-QTLC0-NATIVE-HASH-HOT-PATH-002
  direct native incremental hash intrinsics
  SIMD and hardware acceleration selected by target capability
  tree-parallel Blake3 for large compiler products
  direct byte/string/scalar/aggregate hashing without reflection
  no allocation for fixed digest values
  assembly and executable proofs for every supported algorithm family

POST-QTLC1-HASH-CACHE-MIGRATION-003
  replace string-only CacheHash256 storage with canonical Hash256 bytes
  retain hex only as rendering/path text
  migrate QDB, ProductStore, object, link, ABI, package, and toolchain keys
```

Current implementation sequence:

```text
POST-QTLC1-BYTES-SEED-001: implemented
POST-QTLC1-HASH-OWNED-DIGEST-002: implemented
POST-QTLC1-HASH-BLAKE3-XXH3-003: implemented
POST-QTLC1-HASH-STABLE-COMPILER-004
  status: implemented
  canonical field-framed stable hashing now covers compiler IDs, source spans,
  syntax/declarations, type symbols, HIR, MIR, native object keys, and query keys
  excludes allocation identity, capacity, object padding, and derived cache state
POST-QTLC1-HASH-TYPED-DOMAINS-005
POST-QTLC1-CACHE-HASH-MIGRATION-006
POST-QTLC0-NATIVE-BYTES-HASH-HOT-PATH-007
```

## Indexed Typecheck Lookup

```text
POST-QTLC1-TYPECHECK-SYMBOL-SCOPE-INDEX-008
  status: implemented
  deterministic XXH3 bucket indexes for symbol names and qualified names
  exact string checks on collisions
  usize buckets/rows and Option<usize> links with no negative index sentinel
  while-let collision traversal with direct native Option lowering
  indexed ScopeId + name bindings with lexical parent traversal
  removes repeated full symbol/binding scans from AST typechecking
```
