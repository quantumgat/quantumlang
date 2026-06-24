# Proof Matrix

This file defines the required positive and negative proof cases for the future
compiled package.

## Positive Runtime Proof

The proof aggregate should be a normal value with a readable `valid` view:

```qn
pub OopSurfaceRoleCapabilityProof {
  implRoleValid: bool = false
  implViewRoleValid: bool = false
  contextRoleValid: bool = false
  extractRoleValid: bool = false
  extendRoleValid: bool = false
  multiRoleVisibilityValid: bool = false
  genericRoleValid: bool = false
  variadicPackValid: bool = false
  spreadValid: bool = false
  sameRoleMultiSurfaceValid: bool = false
  explicitSurfaceQualificationValid: bool = false
  directNativeDispatchValid: bool = false
  noDuplicateSymbolValid: bool = false
  moduleBoundaryValid: bool = false
  moduleImportExportValid: bool = false
  negativeFixtureCatalogValid: bool = false
  highLevelApiValid: bool = false
  asyncExtractionValid: bool = false
  builderMutationValid: bool = false
  middlewarePipelineValid: bool = false
  roleObjectConversionValid: bool = false
  associatedRequirementValid: bool = false
  packQueryValid: bool = false
  overloadDefaultNamedArgValid: bool = false
}

impl view OopSurfaceRoleCapabilityProof {
  pub valid: bool => {
    implRoleValid &&
    implViewRoleValid &&
    contextRoleValid &&
    extractRoleValid &&
    extendRoleValid &&
    multiRoleVisibilityValid &&
    genericRoleValid &&
    variadicPackValid &&
    spreadValid &&
    sameRoleMultiSurfaceValid &&
    explicitSurfaceQualificationValid &&
    directNativeDispatchValid &&
    noDuplicateSymbolValid &&
    moduleBoundaryValid &&
    moduleImportExportValid &&
    negativeFixtureCatalogValid &&
    highLevelApiValid &&
    asyncExtractionValid &&
    builderMutationValid &&
    middlewarePipelineValid &&
    roleObjectConversionValid &&
    associatedRequirementValid &&
    packQueryValid &&
    overloadDefaultNamedArgValid
  }
}
```

## Positive Cases

1. `impl Target : Role` satisfies a role with direct methods.
2. `impl view Target : Role` satisfies role getters from view properties.
3. `context Target : Role` satisfies context methods with bare receiver fields.
4. `extract Target : Role` satisfies extractor methods with bare receiver fields.
5. `extend Target : Role` satisfies extension methods.
6. One surface conforms to multiple roles.
7. Every role in a role list has independent visibility.
8. Missing visibility defaults to private.
9. Generic role conformance works: `Role<T>`.
10. Variadic role conformance works: `Role<...Caps>`.
11. Spread works in target types: `HttpCtx<TraceCap, ...Caps>`.
12. Pack constraints work: every `Caps` member must satisfy its bound.
13. A method body can satisfy more than one compatible role requirement.
14. Compatible shared methods do not produce duplicate native symbols.
15. Direct calls use static native lowering.
16. Explicit role-object conversion, if used, has separate representation.
17. Cross-module `pub` surface-role conformance is visible outside the package.
18. Cross-module `internal` surface-role conformance is visible only inside the package.
19. Cross-module `private` surface-role conformance is visible only inside the owner boundary.
20. Surface-qualified calls are accepted for every surface: `impl`, `view`, `context`, `extract`, and `extend`.
21. Async or streaming extractors preserve backpressure and avoid hidden allocation.
22. Builder/mutation flows return explicit values: `withHeader`, `setTrace`, `consumeBody`.
23. Middleware composition supports `before`, `after`, `around`, and `onError`.
24. Explicit role-object conversion is source-visible and uses a separate carrier.
25. Associated role requirements support `type`, `const`, and error/body contracts.
26. Pack queries support uniqueness, typed index lookup, and role-wide predicates.
27. Defaults and named arguments work with overload resolution.
28. Every capability pack member has explicit `HttpCapability` conformance.
29. Multiple blocks for the same target and surface merge deterministically.
30. Duplicate same-surface members are rejected after merge.
31. Associated const/type access has locked surface-qualified syntax.
32. Native proof reads artifact facts, not hard-coded `true` placeholders.
33. Public exports make roles and model types visible across modules.
34. Re-exports preserve canonical identity through facade modules.
35. Import aliases prevent local-name collisions without changing symbol identity.
36. Cross-module conformances are visible only when exported.
37. Cross-module pack queries use canonical imported type identity.
38. Cross-module associated items obey export visibility.
39. Compile-fail fixtures are real invalid `.qn` files, not comment-only notes.
40. `negativeProofCatalog()` registers every compile-fail fixture path and count.

## Negative Compile-Time Cases

These must be separate failing fixtures after qtlc0 supports the syntax.

1. Missing required role method.
2. Wrong return type for a role method.
3. Wrong parameter list for a role method.
4. A view property attempts to satisfy an incompatible role function.
5. Private conformance accessed outside the owning surface boundary.
6. Internal conformance accessed outside the package.
7. `pub` conformance leaks a private target type.
8. `pub` conformance leaks a private role type.
9. Two visible surfaces expose the same unqualified method name and lookup is ambiguous.
10. Duplicate role methods with incompatible signatures.
11. Variadic pack arity mismatch.
12. Spread appears in a non-pack position.
13. Pack bound violation.
14. `extract` body uses receiver fields without a receiver only when no receiver exists.
15. Native backend attempts dynamic dispatch for a direct call.
16. `internal` conformance is accessed from a different package.
17. `private` conformance is accessed outside the owner boundary.
18. Duplicate pack entries are accepted when `Caps.unique` is required.
19. Pack query uses a missing type and does not return a deterministic invalid result.
20. A role object conversion happens implicitly when the source did not request it.
21. Two role surfaces expose the same unqualified method and overload ranking hides the ambiguity.
22. Associated `type` requirement is missing.
23. Associated `const` requirement has the wrong type.
24. Associated error/body contract leaks a private type through a public role.
25. `async extract` or streaming extract hides allocation in a direct call.
26. Builder flow mutates an immutable receiver without returning a new value.
27. Middleware `around` captures a next step with an incompatible target type.
28. Named argument resolution ignores the parameter name and picks a positional overload.
29. Default argument materialization produces a duplicate native symbol.
30. A capability pack member lacks the required role conformance.
31. Two same-surface blocks define the same member with incompatible signatures.
32. Same-surface merge order changes overload resolution.
33. Associated const access is accepted through the wrong surface.
34. Native proof accepts a direct call that uses a role table.
35. Native proof accepts duplicate symbols for one shared method body.
36. Importing a non-exported symbol succeeds.
37. Re-exporting a private type through a public module succeeds.
38. Two unaliased imports with the same local name are accepted.
39. Import alias changes semantic or native symbol identity.
40. Surface-qualified lookup bypasses module visibility.
41. A role conformance exists but is used without being exported.
42. `Caps.indexOf<T>()` changes when imports are reordered.

## Native Proof Requirements

The compiled proof must inspect or assert these backend facts:

```text
ctx.user()                 -> direct native symbol
ctx.valid<CreateUserBody>() -> monomorphized direct native symbol
ctx.sessionData<T>()        -> monomorphized direct native symbol
ctx.path                    -> direct view getter or direct field load
Role lists                  -> no runtime role table
Multiple conformances       -> no duplicate native method symbols
Visibility filtering        -> happens before overload ranking
Named/default arguments     -> compile-time thunk or direct lowering, no metadata dispatch
Pack queries                -> canonical pack index table, no string scan
Role-object conversion      -> explicit source operation only
Associated requirements     -> substituted before native symbol naming
Streaming extractors        -> direct state machine or direct iterator value
Same-surface blocks         -> canonical merge group before duplicate checks
Capability packs            -> every pack member has declared role conformance
Native artifact proof       -> artifact facts drive direct/no-duplicate checks
Import/export identity      -> canonical package/module/declaration ids
Re-export facade            -> no duplicate type identity through facade modules
Import aliases              -> local binding only, not semantic identity
```

The native backend may generate metadata for diagnostics and artifacts, but
direct user calls must not depend on metadata at runtime.

## Future Commands

After conversion from design-only to separated positive/negative/design inputs:

```sh
qtlc/build/compiler/driver/qtlc check qtlang/compiler/tests/language/positive/oopSurfaceRoleCapability --quiet
qtlc/build/compiler/driver/qtlc build qtlang/compiler/tests/language/positive/oopSurfaceRoleCapability --quiet
qtlang/compiler/tests/language/positive/oopSurfaceRoleCapability/build/debug/oop-surface-role-capability-proof
qtlang/compiler/build/debug/qtlc1 check qtlang/compiler/tests/language/positive/oopSurfaceRoleCapability --quiet
qtlang/compiler/build/debug/qtlc1 build qtlang/compiler/tests/language/positive/oopSurfaceRoleCapability --quiet
qtlang/compiler/build/debug/qtlc1 check qtlang/compiler --quiet
qtlang/compiler/build/debug/qtlc1 build qtlang/compiler --quiet
qtlang/compiler/build/debug/qtlc2 check qtlang/compiler --quiet
```

Negative cases must be run by an expected-fail harness from
`qtlang/compiler/tests/language/negative/oopSurfaceRoleCapability`, not through
normal package check/build.
