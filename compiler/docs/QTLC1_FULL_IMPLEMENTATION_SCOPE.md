# qtlc1 Full Implementation Scope

Status: scope lock before frontend implementation

This document is the hard boundary for qtlc1. If another document is unclear,
this page decides what belongs in qtlc1 and what must wait for qtlc2/qtlc3.

The primitive/core type, memory fact, literal typing, operator subset, control
flow, loop, parallel loop, and quantum control-flow scope are locked separately
in:

```text
QTLC1_CORE_TYPE_MEMORY_OPERATOR_SCOPE.md
QTLC1_CONTROL_FLOW_AND_LOOP_SCOPE.md
```

## Purpose

qtlc1 is only the first QuantumLang-authored compiler.

```text
qtlc0 builds qtlc1
qtlc1 builds qtlc2
qtlc2 builds the real toolchain
```

qtlc1 is not the final toolchain. qtlc1 is the compiler bridge from bootstrap
to a real self-hosted compiler.

## Hard Boundary

qtlc1 must implement compiler capability only.

qtlc1 must not implement:

```text
runtime implementation
stdlib implementation
core library migration
HTTP package
network stack
database drivers
plugin runtime or marketplace
package manager
LSP server
formatter
documentation generator
test runner
full async runtime
actor/service framework
final advanced OOP system
macro/derive system
full unsafe/extern ecosystem
```

Those belong after the self-host gate, mostly in qtlc2/qtlc3.

## Stage Ownership

### qtlc0

qtlc0 is the seed compiler.

qtlc0 should support only enough QuantumLang to compile qtlc1 source:

```text
pub Type { ... }
impl Type { ... }
Type(args) = This { ... }
This
this
direct field access inside impl
this.field when explicit access is clearer
this { field = value }
name(args) = expr
name(args) -> T = expr
name => expr
name(args) mut { ... }
name!()
T?
final expression return
```

qtlc0 may accept `self` for old/bootstrap compatibility, but new qtlc1 source
must not use it.

qtlc0 must not become the permanent owner of:

```text
runtime
stdlib
package manager
plugin ecosystem
advanced OOP
macro system
async runtime
```

### qtlc1

qtlc1 is the first self-host compiler source tree.

qtlc1 must become capable of:

```text
checking qtlc1 source
building qtlc2
emitting deterministic compiler artifacts where required
writing/reading compiler interface artifacts
using incremental query products
```

qtlc1 may contain compiler metadata for runtime/stdlib/package concepts only
when the compiler needs those facts. It must not implement those ecosystems.

Compiler-known primitive/core facts are allowed when they are needed for
parsing, type checking, ABI/layout facts, diagnostics, or incremental cache
identity. The allowed qtlc1 core facts are listed in
`QTLC1_CORE_TYPE_MEMORY_OPERATOR_SCOPE.md`.

### qtlc2

qtlc2 is the real self-hosted compiler stage.

qtlc2 starts toolchain assembly:

```text
core package
runtime base
stdlib base
package metadata
plugin metadata
tooling skeletons
```

qtlc2 is where advanced QuantumLang language surfaces can start becoming real
compiler features, after qtlc1 proves self-hosting.

### qtlc3

qtlc3 is where the final product language/toolchain grows:

```text
pub class Type { ... }
pub resource Type { ... }
pub actor Type { ... }
pub service Type { ... }
role/extend role composition
context/extract framework surfaces
macro/derive
full async/runtime integration
full plugin system
package manager
LSP
formatter
docs generator
release toolchain
```

## Mandatory Quantum Style For qtlc1

Use `this`, `This`, and direct field access. Do not write new qtlc1 code with
Rust-style `self`.

Good:

```qn
impl SourceSpan {
  SourceSpan(fileId: StableId, start: usize, end: usize) =
    This { fileId, start, end }

  length() -> usize = end - start

  empty => start == end

  contains(offset: usize) -> bool =
    offset >= start && offset < end

  rename() mut {
    name = "omg"
  }

  findName() -> string {
    name
  }

  testFunc() -> string {
    return name
  }

  shift(delta: usize) =
    this {
      start = start + delta
      end = end + delta
    }
}
```

Avoid:

```qn
impl SourceSpan {
  length(self: SourceSpan) -> usize =
    self.end - self.start
}
```

Rules:

```text
this
  current object inside impl

This
  current type inside impl

field
  direct field access inside impl

this.field
  explicit field access when needed for clarity or disambiguation

self
  legacy compatibility only; do not use in new qtlc1 source
```

## Method Rules

Constructor:

```qn
impl Token<T> {
  Token(kind: T, text: string) =
    This { kind, text }
}
```

Expression body:

```qn
length() -> usize = end - start
```

Conservative inferred expression body:

```qn
emptyText() =
  text == ""
```

Getter/property:

```qn
empty => text == ""
```

Void block method:

```qn
rename(value: string) mut {
  text = value
}
```

Final expression return:

```qn
findName() -> string {
  name
}
```

Explicit return is still allowed:

```qn
findName() -> string {
  return name
}
```

Mutation rule:

```text
method mutates current object
  use mut

method returns updated copy
  use this { ... }
```

Examples:

```qn
rename(value: string) mut {
  name = value
}
```

```qn
renamed(value: string) =
  this { name = value }
```

## Data Shape Rules

Use QuantumLang data shapes:

```qn
pub Type {
  field: T
  private secret: T
}
```

Visibility:

```text
pub Type { field: T }
  public type, public fields by default

pub Type { private field: T }
  public type, private field override

Type { field: T }
  module-private type, private fields by default

Type { pub field: T }
  module-private type, explicit public field
```

Do not use `let` inside data shapes. `let` is for local variables only.

Do not write redundant public fields inside public data shapes:

```qn
pub Type {
  pub field: T
}
```

Prefer:

```qn
pub Type {
  field: T
}
```

## Literal Scope For qtlc1

qtlc1 should support only compiler-needed literal forms:

```text
string literals
integer literals
float literals if compiler source needs them
bool literals
Type { ... } data literals
This { ... } constructor result literals
this { ... } copy/update literals
Vec/list literals if compiler source needs them
fixed Array literals only with target type/context
anonymous record literals
Map literals only with quoted keys or target type/context
T? option shorthand
plain T value lifting into T? when target type is explicit
none
```

Final core literal rules:

```qn
let nums = [1, 2, 3]
```

Default:

```text
[1, 2, 3] -> Vec<i32>
```

Target type can force a fixed array:

```qn
let fixed: Array<i32, 3> = [1, 2, 3]
```

Empty list literals need target type or clear context:

```qn
let tokens: Vec<Token> = []
```

Anonymous records use unquoted field names:

```qn
let test = {
  id: "23452346",
  user: "salam",
}
```

Meaning:

```text
test.id
test.user
inferred type: { id: string, user: string }
```

Quoted-key literals are dynamic maps:

```qn
let table = {
  "main": mainSymbol,
  "App": appSymbol,
}
```

Meaning:

```text
Map<string, Symbol>
```

Typed object/data literals always include the type name:

```qn
let user = User {
  id: "23452346",
  name: "salam",
}
```

Option shorthand:

```qn
let name: string? = "salam"
let missing: string? = none
let bad = none          // rejected: no optional target type
let bad2: string = none // rejected: string is not optional
```

Meaning:

```text
string? == Option<string>
plain string value lifts into string? because the target type is explicit
none is typed absence and requires T?/Option<T> context
```

Record conversion APIs are allowed as a final language/library direction:

```qn
test.toJson()
test.toBson()
test.toBinary()
test.toVec()
test.toMap()
```

For qtlc1, only the compiler model for anonymous records, field access, and
conversion method lookup belongs here. The actual high-performance JSON, BSON,
binary, Vec, Map, allocator, iterator, and codec implementations belong to
qtlc2/qtlc3 core/stdlib/package work.

Do not add package/domain literals to qtlc1 unless qtlc1 compiler source truly
needs them:

```text
json { ... }
bson { ... }
html { ... }
sql { ... }
http route/controller literals
```

Those are stdlib/package/toolchain features for qtlc2/qtlc3.

## qtlc1 Module Responsibilities

### driver

Must implement:

```text
compiler command model
session/config facts
minimal app entry behavior
exit codes
diagnostic mode selection
```

Must not implement:

```text
package manager commands
LSP server
formatter
installer
plugin marketplace CLI
```

### source

Must implement:

```text
SourceText
SourceFile
SourceSpan
LineMap
SourceMap
SourceFileSystem model
content hash facts
encoding and newline facts
offset to line/column mapping
```

Must not implement:

```text
full OS runtime filesystem
watch service
IDE file synchronization
```

### diagnostics

Must implement:

```text
diagnostic codes
severity
labels
diagnostic bag
plain/color rendering policy
stable source locations
```

Must not implement:

```text
LSP diagnostics transport
IDE protocol
web UI
```

### syntax

Must implement:

```text
token kind
token
trivia
literal model
scanner
lexer
green AST
parser for qtlc1 source subset
parser recovery
```

Must not implement:

```text
macro expansion
derive system
full final qtlc3 grammar
runtime package DSLs
```

### moduleSystem

Must implement:

```text
quantum.toml facts
module.toml facts
mod.qn facade facts
import/export graph
package graph
module graph
visibility facts
```

Must not implement:

```text
package registry
package install/update
remote dependency resolver
plugin marketplace
```

### nameResolve

Must implement:

```text
scopes
symbols
imports
exports
qualified names
overload candidate sets
visibility diagnostics
```

Must not implement:

```text
macro-generated name expansion
runtime reflection
dynamic plugin lookup
```

### typeSystem

Must implement:

```text
builtin type catalog
data shape facts
field facts
constructor signatures
function/method signatures
basic expression type checking
minimal generics
effects and capabilities
```

Must not implement:

```text
final class/resource/actor/service system
full role composition
full dependent type system
full macro type expansion
```

### semantic

Must implement:

```text
CheckedFile
CheckedModule
SemanticModel
PublicApi
dependency fingerprints
cached semantic diagnostics
```

Must not implement:

```text
stdlib semantic database
runtime behavior model
package manager behavior
```

### hir and mir

Must implement:

```text
HIR products
MIR products
control-flow facts
lowered body facts
deterministic body fingerprints
```

Must not implement:

```text
full production optimizer
runtime scheduling
async executor
```

### optimize

Must implement:

```text
minimal debug optimization pass manager
pass identity
pass result/fingerprint facts
```

Must not implement:

```text
full release optimizer
profile-guided optimization
large production optimization suite
```

### quantum

Must implement compiler facts for:

```text
qubit
qreg
qresult
qfunc
qany
ownership/no-clone
reset/release policy
measurement effects
entanglement/purity facts
gate/capability facts
target verifier facts
QMIR/QTIR fact model
```

Must not implement:

```text
quantum runtime execution
cloud provider SDK
simulator runtime
hardware job manager
```

### machine and platform

Must implement:

```text
target triple facts
CPU target facts
QPU target facts
data layout
ABI facts
object format facts
OS/embedded platform facts
```

Must not implement:

```text
full OS abstraction library
embedded HAL library
device drivers
```

### backend

Must implement:

```text
native lower/select/register/frame plan products
object-unit product plan
link plan
artifact layout products
QIR/OpenQASM/provider artifact plan products
```

Must not implement:

```text
runtime implementation
stdlib implementation
complete production backend optimization suite
provider SDK runtime
```

### buildSystem

Must implement:

```text
BuildPlan
BuildGraph
IncrementalState
CacheKey
ArtifactStore
OutputLayout
hot-build protocol contracts
magic-fast proof gates
```

Must not implement:

```text
package manager
installer
remote cache service
registry client
```

### query

Must implement:

```text
StableId
Fingerprint
QueryKey
ProductFingerprint
BuildDatabase
ProductStore
dependency edges
source/token/AST products
type/HIR/MIR/native/link products
validation and replay contracts
```

Must not implement:

```text
external database service
distributed build server
remote package cache
```

### interface

Must implement:

```text
.qi declaration output model
.qcap compiler-readable capsule model
.qidx lookup index model
qcap/qidx mmap handle contracts
dependency source avoidance proof
```

Must not implement:

```text
public package registry
IDE documentation server
binary package installer
```

### bootstrap

Must implement:

```text
qtlc0 compatibility facts
stage facts
self-host proof metadata
qtlc0 -> qtlc1 -> qtlc2 proof model
```

Must not implement:

```text
permanent bootstrap-only compiler behavior
runtime or stdlib build ownership
```

## Proof Rule

Every qtlc1 implementation command must pass:

```text
qtlc/build/compiler/driver/qtlc check <touched-module>
qtlc/build/compiler/driver/qtlc check qtlang/compiler
qtlc/build/compiler/driver/qtlc build -j4 qtlang/compiler
```

Docs-only scope changes do not require a compiler build, but must update the
roadmap/index so future implementation follows the locked scope.

## Final qtlc1 Success

qtlc1 is complete only when:

```text
qtlc0 builds qtlc1
qtlc1 checks qtlc1 source
qtlc1 builds qtlc2
qtlc2 checks qtlc1 source
qtlc2 reproduces expected deterministic products
```

Only after that should runtime, stdlib, package manager, plugins, LSP,
formatter, docs generator, and release toolchain work move into the new
QuantumLang-native pipeline.
