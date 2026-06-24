# qtlc1 Directory And Module Contract

Status: active scaffold contract

## Package Rule

```text
one package = one quantum.toml
```

For qtlc1:

```text
qtlang/compiler/quantum.toml
```

No package manifest belongs at `qtlang/` yet. `qtlang/` is only the future
workspace container.

## Root Rule

The package root has:

```text
qtlang/compiler/
  quantum.toml
  mod.qn
  main.qn
```

No root `module.toml` is needed because `quantum.toml` already defines the
package root and `mod.qn` defines the public root surface.

## Module Boundary Rule

Only real submodules get:

```text
module.toml
mod.qn
```

Nested organization folders do not get module files unless they become a true
import boundary.

Good:

```text
quantum/
  module.toml
  mod.qn
  types/
    qubitType.qn
```

Bad:

```text
quantum/types/
  module.toml
  mod.qn
```

unless `types` intentionally becomes a separate module.

## Data Shape Rule

New qtlc1 source should use QuantumLang data shapes:

```qn
pub Type {
  field: T
  private secret: T
}
```

Do not put `let` inside data shapes. Do not write redundant `pub field` inside
`pub Type` unless a compatibility test explicitly requires old syntax.

## Module Manifest Names

`module.toml` names are local names only:

```toml
[module]
name = "backend"
```

Not:

```toml
name = "qtlc1.backend"
```

The full path is derived from package import root plus module graph.

## Reserved Names

Avoid using language keywords or reserved quantum type words as file basenames
or module aliases when the name is only a compiler metadata page.

Reserved quantum words:

```text
qubit
qreg
qresult
qfunc
qany
```

Use compiler metadata names:

```text
qubitType.qn
qregType.qn
qresultType.qn
qfuncType.qn
qanyType.qn
```

Avoid keyword-like module names:

```text
type       use compilerType or types
build      use buildSystem
qif        use quantumIf
qfor       use quantumFor
qwhile     use quantumWhile
qloop      use quantumLoop
```

## Current Top-Level Compiler Modules

```text
driver
source
diagnostics
syntax
moduleSystem
nameResolve
typeSystem
semantic
hir
mir
optimize
quantum
machine
backend
platform
buildSystem
query
interface
bootstrap
```

## Ownership Summary

```text
driver        CLI, command selection, sessions, config
source        source files, spans, line maps, filesystem abstraction
diagnostics   errors, labels, colors, reports, codes
syntax        tokens, lexer, AST, parser, recovery
moduleSystem  manifests, imports, exports, package graph, visibility
nameResolve   scopes, symbols, overload sets, name lookup
typeSystem    type ids, type context, inference, effects, capabilities
semantic      checked files/modules, public API model
hir           high-level lowered representation
mir           mid-level representation and control-flow graph
optimize      compiler optimization pass framework
quantum       first-class quantum language semantics
machine       CPU/QPU target machine facts, ABI, object formats
backend       native and quantum artifact emission
platform      OS and embedded platform facts
buildSystem   build graph, cache keys, output layout, artifact store
query         stable IDs, query keys, dependency edges, cache reports
interface     .qi/.qcap/.qidx surfaces
bootstrap     qtlc0 compatibility and self-host proof metadata
```
