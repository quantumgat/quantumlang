# qtlc1 Implementation Slices

Status: ordered task list for building qtlc2 with qtlc1

Each slice must be independently checkable by qtlc0 before the next slice
starts.

Each slice must also define its cache key and replay product. Fast builds are
part of the compiler architecture, not a later backend-only feature.

Each slice must be justified by qtlc2-builder capability. qtlc1 is complete
only when it can build qtlc2 and prove no-change, one-private-body-change, and
one-public-API-change rebuild behavior.

## Slice 000: Stable IDs And Query Keys

Files:

```text
query/*
```

Deliverables:

```text
StableIdKind
StableId
stable compiler ID wrappers
Fingerprint
ProductFingerprint
QueryKind
QueryKey
DependencyEdge
QueryResult
ProductValidationState
ProductValidation
ProductLookup
QueryExecution
DirtySet
CacheReport
ProductStore
BuildDatabase
```

Proof:

```text
qtlc0 check qtlang/compiler/query
qtlc0 check qtlang/compiler
```

## Slice 001: Source And Span Model

Files:

```text
source/sourceFile.qn
source/sourceText.qn
source/sourceMap.qn
source/span.qn
source/lineMap.qn
source/fileSystem.qn
```

Deliverables:

```text
SourceText owns text
SourceFile owns path + text
SourceSpan stores file/start/end
LineMap maps offsets to line/column
```

Proof:

```text
qtlc0 check qtlang/compiler/source
```

## Slice 002: Diagnostics

Files:

```text
diagnostics/diagnostic.qn
diagnostics/diagnosticCode.qn
diagnostics/diagnosticBag.qn
diagnostics/label.qn
diagnostics/reporter.qn
diagnostics/color.qn
```

Deliverables:

```text
typed diagnostic code
primary/secondary labels
stable severity model
plain and color rendering policy
```

## Slice 003: Token And Lexer

Files:

```text
syntax/token/*
syntax/lexer/*
```

Deliverables:

```text
TokenKind
SyntaxToken
Trivia
Literal
Scanner
Lexer
number/string lexing
error recovery for bad characters and unterminated strings
```

## Slice 004: AST And Parser

Files:

```text
syntax/ast/*
syntax/parser/*
```

Deliverables:

```text
module item parser
function/type/data shape parser
expression parser
statement parser
pattern/type parser
recovery and precedence rules
normal control-flow parser
loop/while/for parser
minimal par-for/shard-for parser facts
qif/qfor/qwhile/qloop parser facts
```

## Slice 005: Module System

Files:

```text
moduleSystem/*
```

Deliverables:

```text
quantum.toml reader model
mod.qn import/export surface model
module graph
visibility facts
package graph
```

## Slice 006: Name Resolution

Files:

```text
nameResolve/*
```

Deliverables:

```text
Symbol
Scope
Resolver
OverloadSet
private/public visibility diagnostics
```

## Slice 007: Type System

Files:

```text
typeSystem/*
```

Deliverables:

```text
Type
TypeId
TypeContext
GenericParameter
TypeInference
TypeChecker
EffectSet
CapabilitySet
```

Do not use file/module name `type`; use `compilerType`.

## Slice 008: Semantic Model

Files:

```text
semantic/*
```

Deliverables:

```text
CheckedFile
CheckedModule
SemanticModel
PublicApi
```

## Slice 009: Quantum Language Model

Files:

```text
quantum/*
```

Deliverables:

```text
qubit/qreg/qresult/qfunc/qany metadata
resource ownership/lifetime/no-clone facts
quantum effects and purity
gate/intrinsic/measurement operation model
circuit/topology/schedule model
quantum control-flow profiles
qif/qfor/qwhile/qloop facts
resource/effect/target verifiers
QMIR/QTIR lowering model
```

## Slice 010: HIR And MIR

Files:

```text
hir/*
mir/*
```

Deliverables:

```text
HirNode
HirModule
AstLowering
MirNode
MirFunction
MirBuilder
ControlFlowGraph
```

## Slice 011: Machine And Platform

Files:

```text
machine/*
platform/*
```

Deliverables:

```text
CPU target triple/machine/features
QPU target/capability/topology
ABI facts
object format facts
OS platform facts
embedded memory/vector/startup/linker profile facts
```

## Slice 012: Backend

Files:

```text
backend/native/*
backend/quantum/*
```

Deliverables:

```text
native lower/select/register/frame/object/link plan
QIR emitter
OpenQASM emitter
provider profiles
shot/noise/simulator/transpile plans
```

## Slice 013: Build System And Interface Artifacts

Files:

```text
buildSystem/*
query/*
interface/*
```

Deliverables:

```text
BuildPlan
BuildGraph
IncrementalState
CacheKey
ArtifactStore
OutputLayout
BuildDatabase
QueryKey
CacheReport
.qi writer
.qcap writer
.qidx writer
API surface model
```

## Slice 014: Bootstrap Proof

Files:

```text
bootstrap/*
```

Deliverables:

```text
qtlc0 compatibility facts
stage facts
self-host proof metadata
```
