# QuantumLang Compiler

`quantumlang` is the public repository for QuantumLang and its qtlc compiler
toolchain.

QuantumLang is a native-first systems language being designed for classical
software, quantum software, compiler construction, and future hybrid
classical-quantum backends. The compiler is built in stages so the language can
prove itself by compiling its own compiler.

```text
qtlc1  ->  qtlc2  ->  self-hosted qtlc
```

## Stage Model

```text
qtlc1
  QuantumLang compiler source.
  Seed compiler source written in QuantumLang.
  Must stay clean, fast, deterministic, and professionally structured.

qtlc2
  Compiler produced by qtlc1.
  Proves qtlc1 can check and build the compiler source.
  Not fully independent until qtlc1 owns complete MIR, object emission, and
  linking.
```

The rule is simple: qtlc1 should not be weakened for temporary bootstrap
constraints. The public compiler design stays centered on the language model
itself.

## Why QuantumLang

QuantumLang is designed around a hard target: one language that can express
native systems code, high-level application APIs, compiler internals, and
quantum programs without turning the core language into a runtime-heavy
framework.

The design priorities are:

- Native speed with predictable lowering.
- Strong static typing and explicit capability boundaries.
- Direct low-level control for systems, ABI, memory, machine, and backend work.
- High-level API expression through `impl`, `impl view`, `context`, `extract`,
  `extend`, roles, generic extensions, and variadic packs.
- First-class quantum architecture instead of a classical-only language with a
  quantum library bolted on later.
- Self-hosting as the proof that the language is strong enough to build serious
  software.
- Centralized compiler models instead of scattered fallback logic.

The goal is not marketing syntax. The goal is a language where advanced API
design still lowers to direct, inspectable native code.

## Current Compiler Capabilities

qtlc1 currently contains compiler ownership for:

- package and module loading;
- source files, spans, text, bytes, and hash utilities;
- lexer, parser, AST, and syntax ownership;
- declaration records and stable declaration identity;
- module identity, import graph, visibility, and re-export policy;
- name resolution and candidate lookup;
- type symbols, type scopes, type registry, typed facts, and diagnostics;
- literal typing and carrier constructor resolution;
- generic substitution and candidate overload resolution;
- HIR and MIR pipeline ownership;
- native backend planning and object/link work;
- quantum type, IR, effect, resource, circuit, transform, and verification
  ownership;
- query, storage, incremental product replay, and build cache direction;
- self-host build orchestration for qtlc1 and qtlc2.

The active language proof direction includes:

- first-class `role` declarations;
- OOP surfaces: `impl`, `impl view`, `context`, `extract`, and `extend`;
- per-role visibility: `pub`, `internal`, `private`, and default private;
- role conformance through explicit conformance records, not structural
  fallback;
- generic extension bounds such as `extend<T: Readable<T>> T`;
- variadic capability packs such as `HttpCtx<...Caps>`;
- typed object literal shorthand such as `const value: Counter = { value: 3 }`;
- method-style text and byte capability APIs such as `text.startsAsciiUpper()`
  and `text.isAsciiLowerAt(index)`.

Some of these surfaces are still being moved into centralized qtlc1
implementation. The roadmap below describes that boundary in public terms.

## Repository Layout

Top-level compiler modules are declared by `module.toml` files. Keep the
top-level module count stable; growth should happen inside ownership folders
first, then become declared submodules only when the public API is ready.

```text
quantumlang/
  compiler/
      backend/         backend planning and native/quantum backend ownership
      bootstrap/       compiler stage product records
      buildSystem/     package build graph, cache, output, self-host flow
      constEval/       compile-time evaluation
      controlFlow/     control-flow facts and analysis
      core/            bytes, text, hash, communication, common primitives
      declaration/     declaration rows and stable identities
      diagnostics/     diagnostic codes, reports, and contracts
      driver/          command entry and compiler orchestration
      frontend/        package frontend product construction
      hir/             high-level IR
      host/            host filesystem/process/native bridge
      interface/       public compiler interface and qcap/qi/qidx APIs
      layout/          layout and ABI planning
      machine/         target, ABI, arch, object, capability models
      mir/             middle IR
      moduleSystem/    modules, imports, visibility, re-exports
      monomorphize/    generic specialization ownership
      nameResolve/     symbol and name lookup
      optimize/        optimization ownership
      package/         package resolution
      platform/        host, OS, embedded, quantum platform models
      quantum/         quantum types, IR, effects, resources, lowering, verify
      query/           query engine and product replay
      semantic/        semantic facts and checked source bridge
      source/          source files, line maps, spans, fingerprints
      storage/         persistent/cache storage
      syntax/          lexer, token, parser, AST
      tests/           compiler and language proof packages
      typeSystem/      type catalog, checking, generic, inference, ownership
```

## Compiler Pipeline

```text
PackageInput
  -> SourceManifest
  -> Syntax tokens and AST
  -> DeclarationRecordTable
  -> ModuleIdentity and import graph
  -> TypeSymbolTable
  -> TypeModuleScopes
  -> TypeScopeTable
  -> OopSurfaceTable
  -> SurfaceConformanceTable
  -> TypeRegistry
  -> TypedNodeFact
  -> TypecheckProduct
  -> HIR
  -> MIR
  -> NativeObjectPlan / QuantumIRPlan
```

The long-term qtlc1 rule is that every major concept has one owner:

- declarations own source identity;
- module system owns visibility and imports;
- type system owns symbols, scopes, generic substitution, and diagnostics;
- OOP surface tables own role conformance and surface lookup;
- query/storage own replay and incremental product reuse;
- backend owns object emission and linking.

## OOP, Roles, And Capability APIs

QuantumLang's advanced API direction is based on explicit surfaces:

```qn
pub HttpCtx<...Caps> {
  request: HttpRequest
}

impl HttpCtx<...Caps> : pub HttpLifecycle<HttpCtx<...Caps>> {
  pub start() -> i64 {
    return 1
  }
}

impl view HttpCtx<...Caps> : pub HttpReadableView<HttpCtx<...Caps>> {
  pub route: string => request.path
}

context HttpCtx<...Caps> : pub CapabilityContext<...Caps> {
  pub hasCaps() -> bool {
    return true
  }
}

extract HttpCtx<...Caps> : pub BodyExtractor<...Caps> {
  pub body<T>() -> Result<T, IntoHttpError> {
    return decode<T>(request.body)
  }
}

extend HttpCtx<...Caps> : pub Authenticated<HttpCtx<...Caps>> {
  pub user() -> AuthUser {
    return AuthUser { id: 1 }
  }
}
```

qtlc1 must lower all of these into one centralized model:

```text
OopSurfaceConformance {
  surfaceKind
  targetType
  roleConformances
  visibility
  generics
  whereBounds
  members
  mergeGroup
}
```

No resolver or typechecker phase should invent its own fallback for role lists,
generic extension lookup, or surface visibility.

## Text And Byte Capability Direction

Public text APIs should be method-style, not child-style free functions:

```qn
const text: string = "Hello"

text.startsUpper()
text.startsAsciiUpper()
text.startsUnicodeUpper()
text.isAsciiLowerAt(1)
text.asciiByteAt(0).isUpper()
```

Naming rule:

```text
isLower()              default text-level behavior, whole value
isAsciiLower()         explicit ASCII behavior, whole value
startsLower()          default first scalar/text behavior
startsAsciiLower()     first byte ASCII behavior
isAsciiLowerAt(index)  indexed byte predicate
asciiByteAt(index)     returns AsciiByte wrapper
```

Bootstrap-safe internals may use direct low-level helpers, but the public qtlc1
API direction is professional owner methods and zero-cost extensions.

## Quantum Direction

QuantumLang is designed so quantum concepts are visible to the compiler, not
hidden inside ordinary library calls.

Planned and active ownership areas include:

- quantum scalar/register types;
- circuit IR;
- quantum operations and gates;
- quantum control-flow validation;
- resource estimation;
- effect tracking;
- simulation backend planning;
- hardware backend integration points;
- classical/quantum boundary diagnostics.

The target is hybrid code where classical control, native backend code, and
quantum lowering can share one compiler pipeline while still preserving the
rules of each domain.

## Build And Check Commands

Commands below are shown from the public compiler repository root.

Check qtlc1 source:

```sh
qtlc check . --quiet
```

Build qtlc1:

```sh
qtlc build .
```

Check qtlc1 with the built qtlc1:

```sh
build/debug/qtlc1 check .
```

Build qtlc2 with qtlc1:

```sh
build/debug/qtlc1 build .
```

Check qtlc1 source with qtlc2:

```sh
build/debug/qtlc2 check .
```

Language proof packages should be checked after `tests/language` is split into
positive, negative, and design inputs:

```sh
build/debug/qtlc1 check tests/language/positive --quiet
```

Negative tests must use an expected-fail harness and must not be part of a
normal package build.

## Public Design Notes

The public design contract for this compiler is intentionally simple:

- keep module ownership explicit;
- keep qtlc1 stronger than temporary compatibility code;
- keep OOP surface roles centralized through one conformance model;
- keep text and byte APIs professional and method-style;
- keep direct user calls lowering to direct native calls;
- keep quantum concepts visible to the compiler pipeline;
- keep language proof packages separated into positive tests, expected-fail
  tests, and design-only material.

This README is self-contained for the public repository.

## Quality Rules

- Do not add fallback behavior that hides incomplete semantics.
- Do not make qtlc1 weaker to avoid temporary toolchain limitations.
- Keep advanced language constructs in the public design instead of replacing
  them with weaker public APIs.
- Keep hot paths indexed and deterministic.
- Keep large files split by ownership, not by random line count.
- Keep public APIs owner-method based where the language supports it.
- Keep design-only files and expected-fail files out of normal build packages.
- Prefer canonical ids over text matching after source rows are constructed.
- Preserve direct native lowering for direct calls.

## Current Roadmap

Immediate roadmap:

1. Split `tests/language/oopSurfaceRoleCapability` into positive buildable
   proof inputs, negative expected-fail fixtures, and design-only notes.
2. Keep the OOP role proof green while adding role lists for every OOP surface.
3. Rebuild qtlc1 with the strengthened toolchain.
4. Implement qtlc1 `OopSurfaceTable` and `SurfaceConformanceTable`.
5. Prove qtlc1 checks language positive tests.
6. Continue splitting `typeSystem/checking` large pages by ownership.
7. Implement hot typed-product replay through query/storage.
8. Complete qtlc1 MIR object emission and native linking so qtlc2 is a complete
   self-host proof.

## License

License is not finalized.

## One-Line Summary

QuantumLang is being built as a native, self-hosting, classical-plus-quantum
language where advanced APIs, compiler-grade safety, and direct backend lowering
are designed together instead of patched together later.
