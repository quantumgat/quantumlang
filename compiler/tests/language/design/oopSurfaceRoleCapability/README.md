# OOP Surface Role Capability Proof

Status: design-only proof package pending split into positive, negative, and
design inputs.

This directory defines the next language proof for QuantumLang OOP. The `.qn`
files in this directory are source-shaped design files, not build inputs yet.
Do not add `quantum.toml`, `module.toml`, or import these files into the main
compiler package until qtlc0 implements the central OOP surface conformance
model. The design intentionally uses future syntax that qtlc0 does not fully
support today.

The qtlc1 implementation roadmap is tracked in:

```text
qtlang/compiler/architecture/oop_surface_role_capability.md
```

## Goal

Prove that every OOP surface can carry role conformances with the same power:

```qn
pub HttpCtx<...Caps> { ... }
impl HttpCtx<...Caps> : pub Omg<HttpCtx<...Caps>> { ... }
impl view HttpCtx<...Caps> : pub OmgView<HttpCtx<...Caps>>, internal DebugView { ... }
context HttpCtx<...Caps> : pub OmgContext<...Caps>, internal TestContext { ... }
extract HttpCtx<...Caps> : pub OmgExtract<...Caps> { ... }
extend HttpCtx<...Caps> : pub OmgExtend<HttpCtx<...Caps>>, private AuditRole { ... }
```

The compiler must treat this as one centralized concept:

```text
OopSurfaceConformance {
  surface: Impl | ImplView | Context | Extract | Extend
  target: HttpCtx<...Caps>
  roles: [role + visibility]
  generics: [...Caps]
  whereBounds: ...
  members: ...
}
```

## Non-Negotiable Requirements

- All surfaces support role lists.
- Role lists support per-role visibility: `pub`, `internal`, `private`, and default private.
- Generic roles and variadic/spread roles work with `HttpCtx<...Caps>`.
- Surface member lookup stays deterministic.
- Same-target and same-surface blocks merge into one deterministic conformance group.
- Every capability pack member has explicit role conformance.
- Import, export, re-export, and aliasing resolve through canonical symbol identity.
- Associated type and const requirements use locked surface-qualified access.
- Direct calls lower to native static calls.
- No hidden allocation, no role table, no reflection, no duplicate native method symbols.
- Explicit role-object conversion is separate from direct calls.
- qtlc1 must not be weakened. If a target proof needs advanced lowering, fix qtlc0 first.

## Files

- `target_syntax.md`: full end-user target syntax and complex API examples.
- `proof_matrix.md`: positive and negative proof cases.
- `importExportCapabilityMatrix.md`: import/export, re-export, alias, canonical identity, and cross-module capability cases.
- `qtlc0_centralized_contract.md`: parser, semantic, type checker, lookup, and native lowering design.
- `roles.qn`: target role declarations, including generic and variadic roles.
- `model.qn`: target HTTP/domain model types used by the proof.
- `httpCtxDesign.qn`: all OOP surfaces on `HttpCtx<...Caps>`.
- `advancedApiDesign.qn`: high-level API proof for defaults, named arguments, streaming extract, builder flow, middleware, associated role requirements, pack queries, and explicit role-object conversion.
- `moduleBoundaryDesign.qn`: cross-module `pub`, `internal`, and `private` surface-role visibility proof.
- `moduleImportExportDesign.qn`: module import/export, re-export, alias, conformance export, and cross-module pack proof.
- `nativeArtifactDesign.qn`: native lowering artifact facts for direct dispatch, explicit role-object dispatch, and duplicate symbol checks.
- `genericSpreadDesign.qn`: generic `T, ...Caps` and spread proof forms.
- `proofDesign.qn`: future runtime proof aggregate and end-user API.
- `negativeDesign.qn`: build-safe catalog for compile-fail fixtures.
- `compileFail/`: real invalid `.qn` files that future test tooling must compile expecting failure.

## Conversion Plan

The next conversion step is to separate input kinds before any broad qtlc1
language-test command runs:

```text
tests/language/
  positive/
    oopSurfaceRoleCapability/
  negative/
    oopSurfaceRoleCapability/
  design/
    oopSurfaceRoleCapability/
```

When qtlc0 is ready, convert the positive directory to a real proof package:

```text
oopSurfaceRoleCapability/
  quantum.toml
  module.toml
  mod.qn
  main.qn
  model/*.qn
  roles/*.qn
  behavior/*.qn
  use/*.qn
```

Expected commands after conversion:

```sh
qtlc/build/compiler/driver/qtlc check qtlang/compiler/tests/language/positive/oopSurfaceRoleCapability --quiet
qtlc/build/compiler/driver/qtlc build qtlang/compiler/tests/language/positive/oopSurfaceRoleCapability --quiet
qtlang/compiler/tests/language/positive/oopSurfaceRoleCapability/build/debug/oop-surface-role-capability-proof
qtlang/compiler/build/debug/qtlc1 check qtlang/compiler/tests/language/positive/oopSurfaceRoleCapability --quiet
qtlang/compiler/build/debug/qtlc1 build qtlang/compiler/tests/language/positive/oopSurfaceRoleCapability --quiet
qtlang/compiler/build/debug/qtlc1 check qtlang/compiler --quiet
qtlang/compiler/build/debug/qtlc1 build qtlang/compiler --quiet
```

Compile-fail commands after qtlc0 supports this surface:

```sh
qtlc/build/compiler/driver/qtlc check qtlang/compiler/tests/language/negative/oopSurfaceRoleCapability/surface/missingRequiredRoleMethod.qn --expect-fail
qtlc/build/compiler/driver/qtlc check qtlang/compiler/tests/language/negative/oopSurfaceRoleCapability/module/importNonExportedSymbol.qn --expect-fail
qtlang/compiler/build/debug/qtlc1 check qtlang/compiler/tests/language/negative/oopSurfaceRoleCapability/surface/missingRequiredRoleMethod.qn --expect-fail
```
