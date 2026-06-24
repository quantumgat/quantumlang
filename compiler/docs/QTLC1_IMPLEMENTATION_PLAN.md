# qtlc1 Implementation Plan

Status: implementation must follow this plan before expanding scope

## Purpose

`qtlc1` is the first QuantumLang-authored compiler. It is built by qtlc0.
Its job is to become strong enough to build qtlc2 and to do that with the
incremental speed model that qtlc2 will inherit.

```text
qtlc0 builds qtlc1
qtlc1 builds qtlc2
qtlc2 builds the real toolchain
```

The main output of qtlc1 is not a final user compiler. The main output is a
working qtlc2 compiler artifact plus proof that qtlc1 can rebuild it quickly
after no-change, private-body-change, and public-API-change edits.

The implementation order for this normal-project build path is defined in
[QTLC1_NORMAL_PROJECT_COMPILATION_IMPL_COMMANDS.md](QTLC1_NORMAL_PROJECT_COMPILATION_IMPL_COMMANDS.md).

The global cache and binary cache-object contracts are implemented. Action
hash and content hash separation is the next normal-project compilation slice.

## Non-Goals For qtlc1

Do not implement the full ecosystem in qtlc1:

```text
runtime implementation
stdlib implementation
HTTP package
plugin marketplace
package manager
LSP server
formatter
docs generator
full async runtime
actor/service framework
final advanced OOP system
```

Those start after qtlc1 can rebuild itself or when qtlc2 begins toolchain
assembly.

## Required qtlc1 Capability

qtlc1 must include enough compiler capability to compile qtlc1 source:

```text
source loading
diagnostics
lexer
parser
AST
module/import/export graph
name resolution
basic type checking
effects and capability facts
semantic checked model
HIR
MIR
minimal optimization pass manager
native backend plan/emission path
interface artifact plan
incremental build/cache metadata model
bootstrap compatibility proof
first-class quantum language model
incremental build/cache contracts from the first source slice
demand-driven query database and product store
stable compiler IDs for symbol/body/function-level invalidation
qcap/qidx metadata loading for dependency source avoidance
qtlcd hot-build protocol plan
```

All required capability is judged by qtlc2-builder value. If a feature is not
needed to parse, check, lower, cache, emit, link, or replay qtlc2 compiler
source, it should wait for qtlc2/qtlc3.

Required qtlc2-builder capability:

```text
qtlc1 can check qtlc2 source
qtlc1 can build qtlc2 native executable/package artifacts
qtlc1 can emit qtlc2 interface artifacts
qtlc1 can reuse unchanged qtlc2 query products
qtlc1 can precisely invalidate one changed body
qtlc1 can precisely invalidate one changed public symbol and its dependents
qtlc1 can avoid dependency source reads through qcap/qidx
qtlc1 can show clear cache hit/miss and changed-product reports
```

## qtlc1 Language Style

qtlc1 source should use the qtlc0-supported clean subset:

```text
pub Type { ... }
impl Type { ... }
pub name(args) -> T { ... }
name(args) -> T { ... }
Type(args) = This { ... }
This
this
field access inside impl
this { field = value }
name(args) = expr
name(args) -> T = expr
name => expr
name(args) mut { ... }
name!()
T?
final expression return
```

Full scope and style rules are locked in:

```text
QTLC1_FULL_IMPLEMENTATION_SCOPE.md
QTLC1_CORE_TYPE_MEMORY_OPERATOR_SCOPE.md
QTLC1_CONTROL_FLOW_AND_LOOP_SCOPE.md
```

New qtlc1 code must prefer:

```text
this
This
direct field access inside impl
this.field only when explicit disambiguation is useful
```

Do not write new qtlc1 code with Rust-style `self`. qtlc0 may continue to
accept `self` only as legacy/bootstrap compatibility.

qtlc1 may use only compiler-needed literals:

```text
primitive literals
Type { ... }
This { ... }
this { ... }
T?
[1, 2, 3] as Vec<T> by default
[] only with target type/context
Array<T, N> target type can force fixed array
{ field: value } anonymous record literal
{ "key": value } dynamic Map literal
none
```

Optional values are target-typed:

```qn
let name: string? = "Ali"
let age: i64? = 30
let missing: string? = none
```

`none` without `T?`/`Option<T>` context is rejected. `none` assigned to a
plain non-optional type is also rejected.

Record literals are structural values:

```qn
let test = {
  id: "23452346",
  user: "salam",
}
```

This is not a map and not a Vec. It is an anonymous record with typed fields:

```text
test.id
test.user
```

Quoted keys create a map:

```qn
let table = {
  "main": mainSymbol,
  "App": appSymbol,
}
```

Package/domain literals such as `json { ... }`, `bson { ... }`, HTTP route
literals, SQL literals, and similar stdlib/package APIs belong to qtlc2/qtlc3,
not qtlc1, unless qtlc1 compiler source truly requires them.

Conversion methods such as `toJson()`, `toBson()`, `toBinary()`, `toVec()`,
and `toMap()` are a final language/library direction. qtlc1 may model method
lookup for these shapes if compiler source needs it, but real codec and
collection implementations belong to qtlc2/qtlc3 core/stdlib/package work.

The qtlc1 primitive/core type list, memory facts, operator scope, and bootstrap
compatibility aliases are locked in:

```text
QTLC1_CORE_TYPE_MEMORY_OPERATOR_SCOPE.md
```

Normal control flow, loops, minimal parallel loop facts, and quantum
control-flow scope are locked in:

```text
QTLC1_CONTROL_FLOW_AND_LOOP_SCOPE.md
```

Data shape visibility is locked to this QuantumLang rule:

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

Do not use `let` inside data shapes. `let` is for local bindings only.
Do not use redundant public fields inside a public data shape:

```text
pub Type { pub field: T }
```

Prefer:

```qn
pub CompilerCommand {
  name: string
  private password: string
}
```

Use `pub struct Type { ... }` only as qtlc0 compatibility syntax when the
short data-shape syntax cannot express the construct yet. New qtlc1 source
should prefer `pub Type { ... }`.

qtlc0 proof fixtures:

```text
qtlc/tests/fixtures/bootstrap_safe_qtlc1_style_ok.qn
  proves constructor return inference, expression-body return inference,
  this/This, implicit field access, this-update, getter arrow, mut methods,
  bang methods, option shorthand, fn-free declarations, and final expressions

qtlc/tests/fixtures/bootstrap_void_return_rule_bad.qn
  proves non-void functions must still return a value on every path
```

Checked separately:

```text
impl view Type { ... }
context Type { ... }
extract Type { ... }
```

These are no longer parser-future-only surfaces. qtlc0 already accepts the
surface shapes, and qtlc1 source can use them. The remaining bootstrap gap is
typed value-producing `impl view` property bodies, especially block/if-else
forms such as:

```text
pub firstName: string => { ... }
pub lastName: string => { ... }
pub fullName: string => { ... }
```

qtlc0 should use the stricter rule:

```text
single-expression impl view property:
  type may be inferred

block-bodied impl view property:
  explicit type required
```

That gap must be fixed in qtlc0 so block-bodied properties resolve to their
declared type instead of collapsing to `void`.

Avoid until qtlc2:

```text
class/resource/actor/service
role/extend role composition
macro/derive system
full unsafe capsule system
```

## Build Stages

### Stage A: Scaffold Proof

Done:

```text
qtlc0 checks each qtlc1 module
qtlc0 checks qtlang/compiler
qtlc0 builds qtlang/compiler scaffold
```

### Stage B: Frontend Seed

Starts only after:

```text
POST-QTLC1-MAGIC-FAST-001
```

Implement:

```text
source
diagnostics
syntax/token
syntax/lexer
syntax/ast
syntax/parser
source/token/AST cache keys
```

Goal:

```text
qtlc1 can parse a small .qn file and emit a stable AST/debug report
qtlc1 can replay unchanged source/token/AST products
```

### Stage C: Semantic Seed

Implement:

```text
moduleSystem
nameResolve
typeSystem
semantic
```

Goal:

```text
qtlc1 can check names, imports, basic types, and root module exports
```

### Stage D: IR Seed

Implement:

```text
hir
mir
optimize pass shell
```

Goal:

```text
qtlc1 can lower checked source into deterministic HIR/MIR facts
```

### Stage E: Native Output Seed

Implement:

```text
machine
backend/native
buildSystem
interface
```

Goal:

```text
qtlc1 can emit package metadata, interface artifacts, native object-unit
products, and a qtlc2 executable/package artifact plan
```

### Stage F: qtlc1 Self-Host Gate

Goal:

```text
qtlc0 -> qtlc1
qtlc1 -> qtlc2
qtlc2 output is deterministic where expected
```

Fast-build proof:

```text
qtlc1 no-change rebuild of qtlc2 replays cached products
qtlc1 private body edit rebuilds only affected body/object/link products
qtlc1 public API edit rebuilds exact exported-symbol dependents
qtlc1 dependency build uses qcap/qidx and avoids source reads
```

Only after this should runtime/stdlib/tooling work move into the new tree.
