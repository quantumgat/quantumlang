# qtlc0 OOP Surface Implementation Commands

Goal: make qtlc0 accept the Quantum-style OOP surfaces needed to write qtlc1 cleanly, while keeping runtime-heavy semantics for qtlc2/qtlc3.

## POST-QTLC0-OOP-SURFACE-DOCS-001

- Historical bootstrap item. The `extend Type as Role` spelling below is
  superseded by `POST-QTLC0-OOP-ROLE-CONFORMANCE-VISIBILITY-004`.
- Lock qtlc0 scope for:
  - `impl view Type { property => expr }`
  - `role Name { ... }`
  - `extend Type as Role { ... }`
  - `context Type { ... }`
  - `extract Type { ... }`
- Rule: qtlc0 implements syntax and basic type/object-model attachment.
- Rule: qtlc0 does not implement full role specialization, dynamic dispatch, HTTP extractor runtime, or capability runtime.

## POST-QTLC0-IMPL-VIEW-001

- Parse `impl view Type { property => expr }`.
- Lower each `property => expr` to a zero-argument getter method.
- Allow property access through existing method/field resolution.
- Keep output zero-cost in native backend.

## POST-QTLC0-CONTEXT-EXTRACT-SURFACE-001

- Parse `context Type { ... }`.
- Parse `extract Type { ... }`.
- Attach contained methods to `Type` like an `impl Type` block.
- Preserve the block surface name in AST text for future qtlc1/qtlc2 policy.

## POST-QTLC0-ROLE-SURFACE-001

- Parse `role Name { ... }` as QuantumLang role syntax.
- Reuse existing trait declaration model internally.
- Accept method signatures without `fn`.
- Keep `trait` compatibility during bootstrap.

## POST-QTLC0-EXTEND-AS-ROLE-001

- Historical bootstrap item. This syntax is now rejected with a migration
  diagnostic.
- Parse `extend Type as Role { ... }`.
- Internally map to the existing trait-impl shape `impl Role for Type`.
- Attach methods to `Type` for direct calls.
- Register role method names for basic contract checking.

## POST-QTLC0-OOP-SURFACE-PROOF-001

- Add qtlc0 proof source covering all five surfaces.
- Build `qtlang/qtlc0_test_build` with native backend.
- Run the proof executable and require exit code `0`.

## POST-QTLC0-OOP-SURFACE-GENERIC-ROLE-PROOF-002

- Add a real module-boundary proof under `qtlang/qtlc0_test_build/src/genericSurface`.
- Export/import the proof through `module.toml` and `mod.qn`.
- Prove generic `impl view SurfaceBox<T>` property access.
- Prove generic `role SurfaceReadable<T>` and `extend SurfaceBox<T> as SurfaceReadable<T>`.
- Prove `context SurfaceBox<T>` and `extract SurfaceBox<T>` attach methods to the generic owner.
- Prove bare field access and explicit `this.field` access both work.
- Build `qtlang/qtlc0_test_build` with native backend and run the proof executable with exit code `0`.

## POST-QTLC0-OOP-SURFACE-VARIADIC-ROLE-PROOF-003

- Add `PackBox<...T>` under the same real module-boundary proof package.
- Add `role Decoder<...T>`.
- Add `extend PackBox<...T> as Decoder<...T>`.
- Add `impl view PackBox<...T>`.
- Add `context PackBox<...T>`.
- Add `extract PackBox<...T>`.
- Prove the variadic OOP surface through `module.toml`, `mod.qn`, export, import, native build, and executable run.
- Keep first proof on scalar/string/bool view and method results; raw pack payload ABI is a separate backend item.

## POST-QTLC0-OOP-ROLE-CONFORMANCE-VISIBILITY-004

- Replace `extend Type as Role` with:
  - `extend Type : pub Role`
  - `extend Type : internal Role`
  - `extend Type : private Role`
  - `extend Type : Role`, where omitted visibility means `private`.
- Allow multiple role conformances in one `extend` block.
- Keep one method-owning block; never clone methods or emit duplicate native
  symbols for multiple roles.
- Store role name and visibility as structured conformance metadata.
- Check every role requirement against the shared method set.
- Reject unknown roles, missing methods, incompatible signatures, duplicate
  role entries, and public visibility leaks.
- Filter cross-module role lookup by conformance visibility before overload
  ranking.
- Keep generic bounds independent from visibility:
  - `extend Box<T> where T: Bound : pub Role<T>`.
- Remove `extend Type as Role` from Quantum-style qtlc1 sources and proofs.
- Add positive, negative, cross-module, generic, and native zero-cost proofs
  under `qtlang/qtlc0_role_visibility_proof`.
- Build the proof project, run it, check negative diagnostics, build
  `qtlang/qtlc0_test_build`, and check `qtlang/compiler`.

## POST-QTLC0-IMPL-VIEW-TYPED-PROPERTY-DOCS-004

- Lock the rule that `impl view` properties are value-producing typed properties, not void statement blocks.
- Allow both inferred and explicit property types:
  - `pub name => expr`
-  `pub name: string => expr`
-  `pub name: string => { ... }`
- Rule: single-expression view properties may omit the type when the expression type is obvious.
- Rule: block-bodied view properties must declare an explicit type.
- Rule: every property body branch must resolve to one final type.
- Rule: this must work for primitive, compiler/core, custom, and instantiated generic types.
- Rule: `impl view` remains read-only surface state; callable behavior stays in `impl`.

## POST-QTLC0-IMPL-VIEW-TYPED-PROPERTY-TYPECHECK-005

- Type-check `impl view` property bodies as expression-valued bodies.
- If a property has an explicit type annotation, require the body to match that type exactly.
- If a property is single-expression and has no explicit type annotation, infer one final type from the expression.
- If a property uses a block body, require an explicit type annotation.
- Reject mixed branch types with a normal type mismatch diagnostic.
- Reject value-less/void property bodies unless the declared property type is `void`.

## POST-QTLC0-IMPL-VIEW-BLOCK-BODY-LOWERING-006

- Treat `pub field => { ... }` as a value-producing block, not as a statement-only block.
- Require an explicit declared property type before lowering the block body.
- Use the final expression of each branch as the property result.
- Support:
  - single expression body
  - block body
  - `if/else` branch bodies
  - nested `if/else`
  - constructor/custom-type results
- Keep native lowering zero-cost by lowering the property to a getter body with direct returns.

## POST-QTLC0-IMPL-VIEW-TYPED-PROPERTY-PROOF-007

- Add qtlc0 fixture coverage for typed `impl view` properties:
  - `bool`
  - `int`, `uint`
  - `byte`
  - signed integers: `i8`, `i16`, `i32`, `i64`, `isize`
  - unsigned integers: `u8`, `u16`, `u32`, `u64`, `usize`
  - floats: `f32`, `f64`
  - `string`
  - custom user type
  - generic instantiated type such as `Option<T>` or `Result<T,E>`
- Prove both:
- explicit type form:
  - `pub name: string => { ... }`
  - `pub ready: bool => { ... }`
- inferred type form for single-expression properties only:
  - `pub empty => count == 0`
  - `pub lastId => firstId + count - 1`
- Prove branch unification and rejection diagnostics for mismatched branch types.
- Prove block-bodied properties without explicit types are rejected with a direct diagnostic.
- Use `qtlang/qtlc0_test_build/src/typedViewPropertyProof.qn` as the source proof file.
- Use `qtlang/qtlc0_typed_view_proof` as the standalone executable proof gate before restoring the same surface inside `qtlang/compiler`.

## POST-QTLC0-IMPL-VIEW-QTLC1-DECLARATION-RANGE-REGRESSION-008

- Add a regression proof matching qtlc1 usage shape:
  - `firstName`
  - `lastName`
  - `fullName`
  - `hasNames`
- Require this exact style to type-check:
  - `pub firstName: string => { ... }`
  - `pub lastName: string => { ... }`
  - `pub fullName: string => { ... }`
  - `pub hasNames: bool => firstName != "" && lastName != ""`
- Require qtlc0 to type these as `string`/`bool`, not `void`.
- Use this as the gate before restoring the final qtlc1 `DeclarationRecordRange` view-only design.

## Later qtlc1/qtlc2 Work

- Role composition: `role HttpContext = RequestContext + ExtractContext`.
- Preserve the locked conformance visibility set: `pub`, `internal`,
  `private`; omitted visibility remains `private`.
- Dynamic role values: `any Role`.
- Static role parameters: `some Role`.
- Framework-specific extractor/runtime semantics.
