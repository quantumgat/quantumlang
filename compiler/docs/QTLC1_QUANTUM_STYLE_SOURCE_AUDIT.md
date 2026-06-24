# qtlc1 Quantum Style Source Audit

Status: source-style migration contract for qtlc1

Goal: qtlc1 source code must be written in QuantumLang style, not as a
bootstrap/Rust-like collection of free helper functions. The compiler should use
the language surfaces by meaning:

```text
pub Type { ... }             data/fact shape
impl Type { ... }            constructors, mutation, copy/update, behavior
impl view Type { ... }       read-only computed properties
role Name { ... }            compiler contract
extend Type as Role { ... }  contract implementation
extend<T: Role<T>> T { ... } bounded helper API
context Type { ... }         scoped compiler state/capability behavior
extract Type { ... }         typed input binding from raw/host data
```

This is not cosmetic. It controls public API shape, incremental fingerprints,
role dispatch, package boundary behavior, and native hot-path lowering.

## Style Rules

Prefer constructor methods over factory helpers:

```qn
pub MirBlock {
  id: i64
  instructionCount: i64
  terminatorKind: string
  valid: bool
}

impl MirBlock {
  pub MirBlock(id: i64, instructionCount: i64, terminatorKind: string) =
    This {
      id
      instructionCount
      terminatorKind
      valid: id >= 0 && instructionCount >= 0 && terminatorKind != ""
    }

  withTerminator(kind: string) =
    this { terminatorKind: kind }
}
```

Avoid this unless the function is genuinely module-level:

```qn
pub mirBlock(id: i64, instructionCount: i64, terminatorKind: string) -> MirBlock {
  return MirBlock { ... }
}
```

Use `impl view` for read-only facts that should feel like fields:

```qn
impl view MirBlock {
  isEntry => id == 0
  isEmpty => instructionCount == 0
  hasTerminator => terminatorKind != ""
}
```

Use `role` only for a real contract with multiple implementations or future
extension points. Do not make roles for one-off data helpers.

Use `context` for compiler state/capability owners. A context block is not just
a nicer impl block; it says this type carries scoped compiler state.

Use `extract` where raw data becomes typed compiler data: CLI args, manifests,
source text, tokens, qcap/qidx metadata, host/environment facts.

## Whole-Compiler Target Layout

```text
source/
  pub data types + impl + impl view
  extract HostSource / package roots / file-system data

syntax/
  pub token/AST facts + impl + impl view
  role ParserPhase or SyntaxProducer where stages are interchangeable
  extract token/source data into parser facts

nameResolve/
  pub scope/symbol facts + impl + impl view
  context Resolver
  role SymbolProvider / ScopeProvider

typeSystem/
  pub type/fact data + impl + impl view
  context TypeCheckContext
  role TypeRule / LiteralTypingRule / ConstraintSolver

semantic/
  pub checked model facts + impl + impl view
  role SemanticPass
  extend checked products as cache/public-api roles

query/
  pub stable IDs/products + impl + impl view
  role QueryProduct<T>, Fingerprinted<T>, Cacheable<T>
  extend<T: QueryProduct<T>> T for cache helpers
  context IncrementalQueryEngine

hir/
  pub HIR facts + impl + impl view
  role AstLowering

mir/
  pub MIR facts + impl + impl view
  role HirLowering / MirTransform

controlFlow/
  pub CFG/loop/reachability facts + impl + impl view
  role ControlFlowRule

optimize/
  role OptimizePass
  extend concrete passes as OptimizePass
  context PassManager

backend/
  role Backend<Target>
  extend NativeBackend as Backend<TargetMachine>
  extend QuantumBackend as Backend<QuantumTarget>

machine/
  pub target/ABI/layout facts + impl + impl view

interface/
  role InterfaceWriter
  extend QiWriter/QcapWriter/QidxWriter as InterfaceWriter
  extract qcap/qidx metadata into typed handles

buildSystem/
  context BuildContext / HotBuildSession
  role ArtifactStore / BuildPlanner
  pub build facts + impl + impl view

driver/
  extract CLI args into CompilerCommand/CompilerConfig
  context CompilerSession
```

## Directory Audit

### `driver`

Current signs:

```text
compilerSessionBootstrap(...)
compilerCliBootstrap()
qtlc1CheckReport(...)
```

Target style:

```qn
context CompilerSession {
  commandName => command.name
  readyExit() = CompilerExitCode(0)
}

extract CompilerHostArgs {
  command() -> CompilerCommand
  config() -> CompilerConfig
}

impl CompilerCommand {
  pub CompilerCommand(name: string, kind: CompilerCommandKind) =
    This { name, kind }
}
```

Needed surfaces:

```text
impl       CompilerCommand, CompilerConfig, CompilerExitCode, Qtlc1CheckReport
impl view  CompilerSession, Qtlc1CheckReport
context    CompilerSession
extract    CompilerHostArgs, CompilerCli
role       CommandRunner later, only when commands become pluggable
```

Priority files:

```text
driver/command.qn
driver/session.qn
driver/hostArgs.qn
driver/cli.qn
driver/checkReport.qn
```

### `source`

Current signs:

```text
sourceReader(...)
sourceReadProduct(...)
qtlc1HostMainSourcePath(...)
qtlc1CompilerHostSourceCheck(...)
```

Target style:

```qn
impl SourceFile {
  pub SourceFile(path: string, byteLength: usize) =
    This { path, byteLength }
}

impl view SourceText {
  empty => byteLength == 0
}

extract HostSourceCheck {
  mainPath() -> string
  hasMainFunction() -> bool
}
```

Needed surfaces:

```text
impl       SourceFile, SourceText, SourceReadRequest, SourceReadProduct
impl view  SourceText, SourceFile, LineMap, SourceSpan
context    SourceReader when it owns filesystem/package-root state
extract    HostSourceCheck, PackageSourceScan
role       SourceProvider / SourceFingerprintRule later
```

Priority files:

```text
source/sourceFile.qn
source/sourceText.qn
source/span.qn
source/sourceReader.qn
source/hostSourceCheck.qn
source/packageSourceScan.qn
```

### `syntax`

Current signs:

```text
lexer(fileId)
lexTokenFact(text)
parseModule(parser)
parseImportItem(parser)
```

Target style:

```qn
context Lexer {
  tokenCount => emittedTokens
}

extract Parser {
  module() -> ParseResult
}

role SyntaxPhase<I, O> {
  run(input: I) -> O
}
```

Needed surfaces:

```text
impl       Token facts, Lexer, Parser, Scanner, AST facts
impl view  token classification, parser/recovery facts, AST shape facts
context    Lexer, Parser, Scanner
extract    Parser for token stream -> AST, Lexer for source -> tokens
role       SyntaxPhase, TokenClassifier, AstBuilder later
```

Priority files:

```text
syntax/lexer/lexer.qn
syntax/lexer/scanner.qn
syntax/parser/parser.qn
syntax/parser/itemParser.qn
syntax/parser/typeParser.qn
syntax/ast/*
syntax/token/*
```

### `nameResolve`

Target style:

```qn
context Resolver {
  currentScope => scope
}

role SymbolProvider {
  resolve(name: string) -> Symbol?
}

extend Scope as SymbolProvider { ... }
```

Needed surfaces:

```text
impl       Symbol, Scope, OverloadSet
impl view  Symbol.isType, Symbol.isFunction, Scope.empty
context    Resolver
role       SymbolProvider, ScopeProvider
extend     Scope/ModuleGraph as symbol providers
```

Priority files:

```text
nameResolve/symbol.qn
nameResolve/scope.qn
nameResolve/resolver.qn
nameResolve/overloadSet.qn
```

### `typeSystem`

Target style:

```qn
context TypeCheckContext {
  currentModule => module
}

role TypeRule {
  check(context: TypeCheckContext) -> DiagnosticBag
}

impl view Type {
  numeric => family == TypeFamily.Numeric
}
```

Needed surfaces:

```text
impl       TypeId, Type, TypeContext, GenericParameter, literal facts
impl view  primitive/core type facts, literal target facts, expression contexts
context    TypeChecker, TypeContext, ExpressionTypingContext
role       TypeRule, LiteralTypingRule, ConstraintSolver, EffectRule
extend     concrete rules as role implementations
extract    type refs from syntax nodes
```

Priority files:

```text
typeSystem/types/*
typeSystem/catalog/*
typeSystem/checking/*
typeSystem/inference/*
typeSystem/literal/*
typeSystem/effect/*
```

### `semantic`

Target style:

```qn
role SemanticPass {
  check(file: CheckedFile) -> DiagnosticBag
}

impl view PublicApi {
  stable => fingerprint != ""
}
```

Needed surfaces:

```text
impl       CheckedFile, CheckedModule, SemanticModel, PublicApi
impl view  public API state and dependency fingerprints
role       SemanticPass, PublicApiContributor
extend     checked products as query/cache roles
```

Priority files:

```text
semantic/checkedFile.qn
semantic/checkedModule.qn
semantic/semanticModel.qn
semantic/publicApi.qn
```

### `query`

Current signs:

```text
QueryKey.init(...)
QueryKey.nativeObject(...)
incrementalQueryEngine(...)
incrementalCleanPlan(...)
```

Target style:

```qn
pub role QueryProduct<T> {
  fingerprint() -> ProductFingerprint
}

pub extend<T: QueryProduct<T>> T {
  cacheKey() = QueryKey.product(this.fingerprint())
}

context IncrementalQueryEngine {
  clean => dirtyCount == 0
}
```

Needed surfaces:

```text
impl       StableId, QueryKey, Fingerprint, product records
impl view  cache hit/miss/readiness facts
role       QueryProduct<T>, Cacheable<T>, Fingerprinted<T>
extend     bounded helper APIs for query products
context    IncrementalQueryEngine, ProductStore
extract    stored product metadata into typed products
```

Priority files:

```text
query/stableId.qn
query/queryKey.qn
query/fingerprint.qn
query/*Product.qn
query/incrementalQueryEngine.qn
query/productStore.qn
```

### `hir`

Target style:

```qn
role AstLowering {
  lower(module: AstModule) -> HirModule
}

impl view HirFunction {
  empty => bodyCount == 0
}
```

Needed surfaces:

```text
impl       HIR node/module/function/block/stmt/expr/type facts
impl view  read-only shape and validity facts
role       AstLowering
context    HirLoweringContext later
extract    AST facts into HIR facts
```

Priority files:

```text
hir/hirNode.qn
hir/hirModule.qn
hir/hirFunction.qn
hir/hirBlock.qn
hir/lowerAst.qn
```

### `mir`

Current style is already partly close after recent cleanups.

Target style:

```qn
impl MirBlock {
  isEntry() = id == 0
  withTerminator(kind: string) =
    this { terminatorKind: kind }
}

impl view MirTerminator {
  returnsValue => returns
}
```

Needed surfaces:

```text
impl       MirBlock, MirBody, MirFunction, MirInstruction, MirTerminator
impl view  CFG/readiness/terminator facts
role       MirTransform, HirLowering
context    MirBuilder
extend     concrete lowering/transform passes
```

Priority files:

```text
mir/mirBlock.qn
mir/mirTerminator.qn
mir/mirBody.qn
mir/mirBuilder.qn
mir/hirMirLoweringFact.qn
```

### `controlFlow`

Target style:

```qn
role ControlFlowRule {
  check(region: ControlFlowRegion) -> DiagnosticBag
}

impl view LoopFact {
  bounded => maxIterations >= 0
}
```

Needed surfaces:

```text
impl       CFG, loop, reachability, require, parallel facts
impl view  read-only proof facts
role       ControlFlowRule, LoopRule, ReachabilityRule
context    ControlFlowBuilder/Analyzer later
extract    syntax loop/control-flow nodes into facts
```

Priority files:

```text
controlFlow/controlFlowGraph.qn
controlFlow/loopFact.qn
controlFlow/reachability.qn
controlFlow/definiteReturn.qn
controlFlow/requireFact.qn
```

### `optimize`

Current signs:

```text
constFoldPass(...)
deadCodePass(...)
inlinePass(...)
passManager(...)
```

Target style:

```qn
pub role OptimizePass {
  name() -> string
  run(module: MirModule) -> MirModule
}

extend ConstFoldPass as OptimizePass { ... }
extend DeadCodePass as OptimizePass { ... }

context PassManager {
  enabledPasses => passCount
}
```

Needed surfaces:

```text
impl       individual pass facts
impl view  pass safety/changed-semantics facts
role       OptimizePass
extend     each pass as OptimizePass
context    PassManager
```

Priority files:

```text
optimize/pass.qn
optimize/passManager.qn
optimize/constFold.qn
optimize/deadCode.qn
optimize/inline.qn
```

### `backend`

Target style:

```qn
pub role Backend<Target> {
  lower(module: MirModule, target: Target) -> BackendArtifact
}

extend NativeBackend as Backend<TargetMachine> { ... }
extend QuantumBackend as Backend<QuantumTarget> { ... }
```

Needed surfaces:

```text
impl       BackendArtifact, native/quantum backend facts
impl view  artifact validity/path/fingerprint facts
role       Backend<Target>, ObjectEmitter, Linker, QuantumEmitter
extend     NativeBackend/QuantumBackend role implementations
context    NativeEmitContext, QuantumEmitContext later
extract    target/profile/backend config
```

Priority files:

```text
backend/backend.qn
backend/artifact.qn
backend/native/*
backend/quantum/*
```

### `machine`

Target style:

```qn
impl view TargetTriple {
  linux => os == "linux"
  bareMetal => os == "none"
}
```

Needed surfaces:

```text
impl       target triple, ABI, layout, object format, arch facts
impl view  target capability facts
role       TargetAbi, ObjectFormat, RegisterModel later if interchangeable
extract    target triple text into typed target facts
```

Priority files:

```text
machine/targetTriple.qn
machine/targetMachine.qn
machine/targetAbiFact.qn
machine/dataLayout.qn
machine/object/*
machine/arch/*
machine/abi/*
```

### `interface`

Current signs:

```text
qiLayout(...)
qiFile(...)
qiWriter(...)
qcapWriter(...)
qidxEntry(...)
```

Target style:

```qn
pub role InterfaceWriter {
  write(surface: ApiSurface) -> InterfaceProduct
}

extend QiWriter as InterfaceWriter { ... }
extend QcapWriter as InterfaceWriter { ... }
extend QidxWriter as InterfaceWriter { ... }
```

Needed surfaces:

```text
impl       API surface, qi/qcap/qidx products and handles
impl view  path validity, stable hash, section layout facts
role       InterfaceWriter, CapsuleReader, IndexReader
extend     QiWriter/QcapWriter/QidxWriter role implementations
extract    stored .qi/.qcap/.qidx metadata into typed handles
```

Priority files:

```text
interface/apiSurface.qn
interface/qiLayout.qn
interface/qiWriter.qn
interface/qcapWriter.qn
interface/qidxWriter.qn
interface/qcapHandle.qn
interface/qidxHandle.qn
```

### `moduleSystem`

Target style:

```qn
impl view Manifest {
  libraryPackage => importRoot != ""
}

extract Manifest {
  fromToml(path: string) -> Manifest
}
```

Needed surfaces:

```text
impl       Manifest, ModuleGraph, PackageGraph, ImportPath, ExportSurface
impl view  module/package boundary and visibility facts
role       PackageResolver, ExportSurfaceProvider
context    ModuleResolutionContext later
extract    quantum.toml/module.toml into Manifest/ModuleGraph
```

Priority files:

```text
moduleSystem/manifest.qn
moduleSystem/moduleGraph.qn
moduleSystem/packageGraph.qn
moduleSystem/importPath.qn
moduleSystem/exportSurface.qn
```

### `buildSystem`

Current signs:

```text
magicFastResult(...)
buildPlan(...)
artifactStore(...)
qtlcdState(...)
```

Target style:

```qn
context HotBuildSession {
  daemonHot => qtlcd.ready
}

pub role BuildPlanner {
  plan(request: HotBuildRequest) -> BuildPlan
}
```

Needed surfaces:

```text
impl       build plan/result/request/response/cache facts
impl view  hot/cold/cache-hit/needs-link facts
role       BuildPlanner, ArtifactStore, CacheKeyProvider
extend     build products as cache/query roles
context    HotBuildSession, BuildContext, qtlcdState
extract    CLI/package manifest into build request
```

Priority files:

```text
buildSystem/buildPlan.qn
buildSystem/magicFastResult.qn
buildSystem/hotBuildRequest.qn
buildSystem/hotBuildResponse.qn
buildSystem/qtlcdState.qn
buildSystem/artifactStore.qn
```

### `diagnostics`

Target style:

```qn
pub role DiagnosticRenderable {
  render(policy: DiagnosticPolicy) -> string
}

extend Diagnostic as DiagnosticRenderable { ... }

context DiagnosticReporter {
  colorEnabled => policy.color.enabled
}
```

Needed surfaces:

```text
impl       Diagnostic, DiagnosticBag, DiagnosticCode, Label, Policy
impl view  severity/code/renderability facts
role       DiagnosticRenderable, DiagnosticSink
extend     Diagnostic/DiagnosticBag role implementations
context    DiagnosticReporter
```

Priority files:

```text
diagnostics/diagnostic.qn
diagnostics/diagnosticBag.qn
diagnostics/reporter.qn
diagnostics/diagnosticPolicy.qn
diagnostics/label.qn
```

### `quantum`

Target style:

```qn
pub role QuantumVerifierPass {
  verify(scope: QuantumScopeFact) -> DiagnosticBag
}

context QuantumLoweringContext { ... }

impl view QuantumScopeFact {
  requiresMeasurement => kind == QuantumScopeKind.Measurement
}
```

Needed surfaces:

```text
impl       quantum scope/type/effect/resource/control-flow facts
impl view  scope/resource/control-flow validity
role       QuantumVerifierPass, QuantumLoweringPass, QuantumProvider
extend     provider/backend profiles as roles
context    QuantumLoweringContext, QuantumVerifyContext
extract    quantum syntax/control-flow into quantum facts
```

Priority files:

```text
quantum/quantumScopeFact.qn
quantum/types/*
quantum/effect/*
quantum/controlFlow/*
quantum/resource/*
quantum/lower/*
quantum/verify/*
```

### `platform`

Target style:

```qn
role PlatformProfile {
  targetTriple() -> TargetTriple
}

extend LinuxPlatform as PlatformProfile { ... }
extend WindowsPlatform as PlatformProfile { ... }
```

Needed surfaces:

```text
impl       platform/OS/embedded facts
impl view  supported/object-format/runtime facts
role       PlatformProfile, EmbeddedBoardProfile
extend     each platform as PlatformProfile
extract    target config into platform facts
```

Priority files:

```text
platform/os/*
platform/embedded/*
```

### `bootstrap`

Target style:

```qn
impl view BootstrapStage {
  selfHosted => stageName == "qtlc1"
}
```

Needed surfaces:

```text
impl       bootstrap compatibility facts
impl view  stage readiness facts
context    BootstrapBuildContext only if stateful
extract    qtlc0 outputs into qtlc1 stage facts
```

Priority files:

```text
bootstrap/stage.qn
bootstrap/qtlc0Compat.qn
```

## Current Anti-Patterns To Remove

Factory helper when constructor is better:

```text
pub targetTriple(...) -> TargetTriple
pub qiLayout(...) -> QiLayout
pub sourceReader(...) -> SourceReader
pub passManager(...) -> PassManager
pub quantumBackend(...) -> QuantumBackend
```

Replace with:

```text
impl TargetTriple { pub TargetTriple(...) = This { ... } }
impl QiLayout { pub QiLayout(...) = This { ... } }
impl SourceReader { pub SourceReader(...) = This { ... } }
```

Standalone validity helpers when view is better:

```text
valid: ...
ready: ...
supported: ...
```

Keep the stored field only if it is a cached product fact. Otherwise prefer:

```qn
impl view Type {
  valid => ...
  ready => ...
  supported => ...
}
```

Bootstrap names that encode qtlc1 host proof should move to `extract` or tests:

```text
qtlc1HostMainSourcePath
qtlc1HostMainLexerTokenCount
qtlc1HostMainParserFunctionCount
```

These are host-proof adapters, not core compiler APIs.

## Migration Priority

### POST-QTLC1-SOURCE-QUANTUM-STYLE-AUDIT-001

Done by this document.

### POST-QTLC1-SOURCE-CONSTRUCTOR-CLEANUP-001

Convert obvious factory helpers to constructors in low-risk fact modules:

```text
driver
diagnostics
source
machine
interface
mir
controlFlow
```

Do not change behavior. Keep compatibility helper functions only where tests or
qtlc0 limitations still require them, and mark them as temporary.

### POST-QTLC1-SOURCE-VIEW-PROPERTIES-001

Move read-only status/validity helpers into `impl view`:

```text
SourceSpan.length/empty
Diagnostic.isError
MirBlock.isEntry/isEmpty/hasTerminator
QueryProduct.changedFrom
BuildPlan.needsLink
TargetTriple.linux/bareMetal
```

Bootstrap note:

```text
qtlc1 source should prefer final property-style surfaces such as:
  pub firstName: string => { ... }
  pub fullName: string => { ... }

If qtlc0 rejects these as void-valued blocks, fix qtlc0 typed impl-view body
resolution instead of weakening qtlc1 back to helper methods.

Single-expression view properties may still omit the type:
  pub empty => count == 0
  pub lastId => firstId + count - 1

Block-bodied view properties should declare the type explicitly:
  pub firstName: string => { ... }
  pub ready: bool => { ... }
```

### POST-QTLC1-SOURCE-CONTEXT-EXTRACT-001

Introduce source-level `context` and `extract` where the meaning is clear:

```text
context CompilerSession
context SourceReader
context Resolver
context TypeChecker
context IncrementalQueryEngine
context PassManager

extract CompilerHostArgs
extract Manifest
extract SourceReadRequest
extract Parser
extract qcap/qidx metadata handles
```

### POST-QTLC1-SOURCE-ROLE-BOUNDARIES-001

Add real contracts only where multiple implementations exist or are planned:

```text
Backend<Target>
InterfaceWriter
OptimizePass
QueryProduct<T>
Cacheable<T>
SourceProvider
SymbolProvider
TypeRule
ControlFlowRule
QuantumVerifierPass
PlatformProfile
```

### POST-QTLC1-SOURCE-EXTEND-AS-ROLE-001

Attach implementations:

```text
NativeBackend as Backend<TargetMachine>
QuantumBackend as Backend<QuantumTarget>
QiWriter/QcapWriter/QidxWriter as InterfaceWriter
ConstFoldPass/DeadCodePass/InlinePass as OptimizePass
Linux/Windows/Macos/NoOs platforms as PlatformProfile
```

### POST-QTLC1-SOURCE-BOUNDED-EXTENSIONS-001

Add bounded helper APIs:

```qn
extend<T: QueryProduct<T>> T { cacheKey() = ... }
extend<T: DiagnosticRenderable<T>> T { render() = ... }
extend<T: Cacheable<T>> T { cacheFingerprint() = ... }
```

No unbounded public `extend<T> T` is allowed.

## Proof Requirements

Every migration step must keep:

```text
qtlc/build/compiler/driver/qtlc check -j8 qtlang/compiler
qtlc/build/compiler/driver/qtlc build -j8 qtlang/compiler
```

For each converted module, add or update tests under:

```text
qtlang/compiler/tests/<module>/
```

Proofs must show:

```text
module export/import remains clean
constructor calls still lower directly
view properties are zero-arg getters
role calls are statically dispatched unless explicitly `any Role`
context/extract do not create runtime cost by default
public API fingerprint changes only when public surface changes
```

## Non-Goals For qtlc1 Source Cleanup

Do not implement these as part of the style cleanup:

```text
full dynamic `any Role`
HTTP framework extractor runtime
full capability runtime
automatic JSON/BSON/Binary codec generation
class/resource/actor/service final runtime model
macro/derive system
```

Those belong to qtlc2/qtlc3 after qtlc1 can self-host cleanly.
