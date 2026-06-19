# qtlc0 Centralized Contract

qtlc0 must not implement this feature as five separate systems. The correct
model is one centralized OOP surface conformance pipeline with small
surface-specific policy.

## Canonical Model

```cpp
enum class OopSurfaceKind {
  Impl,
  ImplView,
  Context,
  Extract,
  Extend,
};

enum class OopAssociatedItemKind {
  Type,
  Const,
  Error,
};

struct OopDefaultArgument {
  ParamId parameter;
  ExprRef expression;
  SourceRange range;
};

struct OopSurfacePolicy {
  bool async_extract;
  bool receiver_fields_enabled;
  bool extension_lookup_enabled;
  bool view_property_getters_enabled;
  ReceiverOwnership ownership;
};

struct OopAssociatedItem {
  OopAssociatedItemKind kind;
  Identifier name;
  TypeRef type;
  ExprRef value;
  Visibility visibility;
  SourceRange range;
};

struct OopRoleConformance {
  Visibility visibility;
  TypeRef role;
  SourceRange range;
};

struct OopSurfaceMember {
  Identifier name;
  Visibility visibility;
  FunctionSignature signature;
  std::vector<ArgumentLabel> labels;
  std::vector<OopDefaultArgument> defaults;
  SourceRange range;
};

struct OopSurfaceConformance {
  OopSurfaceKind surface;
  TypeRef target;
  PackageId package;
  ModuleId module;
  MergeGroupId merge_group;
  GenericParamList generic_params;
  VariadicPackList packs;
  BoundList where_bounds;
  OopSurfacePolicy policy;
  std::vector<OopRoleConformance> roles;
  std::vector<OopAssociatedItem> associated_items;
  std::vector<OopSurfaceMember> members;
  SourceRange range;
};

struct ModuleSymbolRow {
  PackageId package;
  ModuleId module;
  DeclId declaration;
  Identifier exported_name;
  Visibility visibility;
  SymbolKind kind;
  SourceRange range;
};

struct ExportRow {
  ModuleId from_module;
  DeclId declaration;
  Identifier exported_name;
  Visibility visibility;
  SourceRange range;
};

struct ImportBinding {
  ModuleId into_module;
  ModuleId from_module;
  DeclId declaration;
  Identifier imported_name;
  Identifier alias;
  Visibility visibility;
  SourceRange range;
};

struct ReExportRow {
  ModuleId facade_module;
  ModuleId source_module;
  DeclId declaration;
  Identifier exported_name;
  SourceRange range;
};

struct ConformanceExportRow {
  ModuleId module;
  MergeGroupId conformance;
  TypeRef target;
  TypeRef role;
  Visibility visibility;
  SourceRange range;
};

struct AssociatedItemExportRow {
  ModuleId module;
  MergeGroupId conformance;
  Identifier item_name;
  OopAssociatedItemKind kind;
  Visibility visibility;
  SourceRange range;
};
```

## Parser Contract

All surface parsers should accept the same role-list grammar:

```text
surface Target generic-params? role-list? where-clause? body

role-list:
  ":" role-entry ("," role-entry)*

role-entry:
  visibility? TypeRef

visibility:
  "pub" | "internal" | "private"
```

Supported forms:

```qn
impl T : RoleA, pub RoleB { ... }
impl view T : RoleA, internal RoleB { ... }
context T : RoleA, pub RoleB { ... }
extract T : RoleA, pub RoleB { ... }
async extract T : RoleA, pub RoleB { ... }
extend T : RoleA, pub RoleB { ... }
```

Parser output should not encode separate special cases. It should produce an
`ImplItem` or a future `OopSurfaceItem` that contains:

```text
surface kind
target type
role entries
members
where clause
generic params
surface policy
associated items
default arguments
module import rows
module export rows
module re-export rows
```

## Semantic Contract

Semantic building must normalize every surface into `OopSurfaceConformance`.

Rules:

- Target type is canonicalized once.
- Role type is canonicalized once.
- Pack arguments are retained, not flattened into text.
- Role visibility is stored per role.
- Surface kind is stored on the conformance.
- Package and module ownership are stored on the conformance.
- Same-target and same-surface blocks share one merge group.
- Default arguments are stored as structured expressions, not lowered strings.
- Associated items are stored beside members on the same conformance row.
- Async extraction is a policy flag on `Extract`, not a separate ad hoc parser path.
- Member source ownership points back to the conformance.
- Constructor members are still owned by the target type, not by a role.
- Imports bind local names to canonical declaration ids.
- Exports expose canonical declaration ids, not copied declarations.
- Re-exports preserve the original canonical declaration id.
- Import aliases are local bindings only and never semantic identity.

## Type Checker Contract

The type checker validates the same record for every surface:

```text
validateTargetReachability
validateGenericParams
validatePackBounds
validateRoleVisibility
validateRoleRequirements
validateAssociatedItems
validateDefaultArguments
validateMemberBodies
validateDuplicateMembers
validateSurfaceMerge
validateImportBindings
validateExportReachability
validateReExportVisibility
validateConformanceExports
validateAssociatedItemExports
validatePublicLeakRules
validateExplicitRoleObjectConversion
```

Surface-specific policy is narrow:

```text
Impl      -> normal functions and constructors.
ImplView  -> view properties may satisfy zero-argument role getters.
Context   -> receiver and capability lookup are enabled.
Extract   -> receiver and extractor lookup are enabled.
Extend    -> extension lookup is enabled.
```

Everything else is shared.

## Same-Surface Merge Contract

Multiple blocks for the same target and same surface are allowed only when they
merge into one deterministic conformance group:

```text
merge key:
  target type hash
  surface kind
  package/module owner
  generic parameter identity
  variadic pack identity
  where-bound identity
```

Rules:

- Merge order is source-order stable but semantically order-independent.
- Duplicate member names are checked after default/named argument normalization.
- Compatible role requirements may share one method body.
- Incompatible duplicate signatures are compile errors.
- A later block cannot weaken visibility of an already public conformance.
- Native symbol naming uses the merge group id, not the physical source block id.

## Module Import/Export Contract

Module resolution must run before OOP surface lookup and must produce canonical
symbol rows.

```text
canonical symbol identity:
  package id
  module id
  declaration id
  generic substitution key
```

Rules:

- `export` exposes a canonical declaration id from the declaring module.
- `import` creates a local binding to that canonical declaration id.
- `import ... as Alias` changes only the local binding name.
- `re-export` forwards the same canonical declaration id through a facade module.
- A conformance is visible across modules only when its conformance export row is visible.
- Associated items are visible across modules only when their conformance and item export rows are visible.
- Public exports cannot expose private target types, private roles, or private associated item types.
- Surface-qualified lookup cannot bypass module visibility.
- Pack queries use canonical imported type ids, not printed names or aliases.

Example facade:

```qn
module http

import http.capability as capability
import http.core as core

export {
  capability.*
  core.{HttpCtx, CapSlots}
}
```

Example alias:

```qn
import http.model.{Session as HttpSession}
import db.session.{Session as DbSession}
```

`HttpSession` and `DbSession` are different local names bound to different
canonical declaration ids. Native symbol naming must use the canonical ids, not
the aliases.

## Lookup Contract

Lookup indexes should use one key shape:

```text
target type hash
member name
surface kind
role id
visibility
module/package owner
generic substitution key
```

Unqualified lookup:

1. Resolve target type.
2. Collect visible conformances for that target.
3. Filter by visibility before overload ranking.
4. Filter by member name and argument shape.
5. Reject ambiguity for unqualified lookup; accept it only when the source uses an explicit surface qualifier.

Surface-qualified lookup is required:

```qn
ctx.impl.reset()
ctx.view.path
ctx.context.require<AuthUser>()
ctx.extract.body<CreateUserBody>()
ctx.extend.user()
```

Ambiguous cross-surface names are compile errors. Explicit surface-qualified
syntax is the required way to select a specific visible surface.

## Native Lowering Contract

Direct calls must lower statically:

```text
ctx.user()                  -> qnn_HttpCtx_AuthUser_user(...)
ctx.body<CreateUserBody>()  -> qnn_HttpCtx_body_CreateUserBody(...)
ctx.path                    -> direct getter or field load
```

Forbidden for direct calls:

- Runtime role table.
- Hidden allocation.
- Reflection lookup.
- String-based method dispatch.
- Duplicate native symbols for one method body satisfying multiple roles.

Allowed only for explicit role objects:

- Role-object carrier.
- Explicit vtable-like representation.
- Dynamic dispatch when requested by the source type.

## Compile-Fail Fixture Contract

Negative proof cases live as real invalid `.qn` files under `compileFail/`.
They are not imported into the positive proof package.

qtlc0 test tooling must support expected-failure checks:

```text
check compileFail/path/to/case.qn --expect-fail
```

Rules:

- Each fixture is a normal source file, not a comment block.
- Each fixture must fail during the expected compiler phase.
- A fixture that succeeds is a test failure.
- A fixture that fails for an unrelated parser crash is a test failure.
- Future diagnostics should record expected diagnostic category and primary span.
- The positive proof may include a catalog of fixture paths, but it must not import invalid files.

## Advanced API Contract

The high-level API proof adds features that must still enter through the same
centralized surface model.

### Defaults And Named Arguments

Default and named arguments must be represented before overload ranking:

```text
call target
argument labels
argument values
defaulted parameter mask
generic substitution key
surface kind
```

Rules:

- Named arguments participate in overload identity.
- Default materialization must not create duplicate native symbols.
- Direct calls still lower statically after defaults are applied.

### Associated Role Requirements

Roles may require associated items:

```qn
role HttpAssociatedSpec<T> {
  type Body
  type Error
  const requiresAuth: bool
}
```

The checker must validate associated `type`, `const`, and error/body contracts
with visibility before native symbol naming. Public associated items must not
leak private types.

### Pack Queries

Variadic packs need canonical metadata available at compile time:

```text
Caps.count
Caps.unique
Caps.has<T>()
Caps.indexOf<T>()
Caps.all<Role>()
```

`Caps.unique` should reject duplicate capability entries when a surface requires
unique capabilities. `Caps.indexOf<T>()` must return a deterministic typed index,
not a string search.

### Async And Streaming Extract

Streaming extraction is a surface policy on `extract`, not a separate compiler
feature:

```qn
async extract HttpCtx<...Caps> : BodyStreamRole<...Caps> { ... }
```

Direct calls must lower to direct state-machine or iterator values. Hidden heap
allocation is forbidden unless the source type explicitly requests an allocated
stream carrier.

### Builder And Middleware Flow

Builder and middleware methods are ordinary surface members:

```qn
impl T : MutableBuilder<T> { ... }
extend T : Middleware<T> { ... }
```

The checker must enforce receiver mutability and return ownership. Immutable
receivers cannot be mutated in place; they must return a new value.

### Role Object Conversion

Role object conversion is explicit source syntax:

```qn
let auth: Authenticated<HttpCtx<AuthUser>> = ctx
```

It may allocate or build a dynamic carrier only because the source requested a
role object. A normal direct call like `ctx.user()` must never take this path.

### Module Boundary

Every conformance row stores package/module ownership:

```text
declaring package
declaring module
target visibility
role visibility
surface visibility
member visibility
```

Visibility filtering happens before overload ranking and before implicit
surface lookup. This is required for `pub`, `internal`, and `private` role lists
to behave consistently across modules.

## qtlc0 Implementation Order

1. Add central OOP surface record type with package/module ownership, merge groups, associated items, defaults, and policy.
2. Add module symbol, export, import, re-export, conformance export, and associated item export rows.
3. Parse role lists for every OOP surface.
4. Preserve current `extend` behavior by mapping it into the central record.
5. Map current `impl`, `impl view`, `context`, and `extract` into the central record.
6. Add role requirement checking for each surface.
7. Add view-property-to-getter satisfaction.
8. Add visibility filtering for surface roles and module exports.
9. Add generic and variadic pack substitution using canonical imported symbol ids.
10. Add duplicate native symbol guard.
11. Add backend direct-lowering assertions.
12. Add default and named argument normalization before overload ranking.
13. Add associated role requirement checking.
14. Add canonical pack query operations.
15. Add explicit role-object conversion lowering.
16. Add async/streaming extract lowering through the extract surface policy.
17. Add builder/middleware receiver ownership validation.
18. Add deterministic same-surface merge and duplicate-member diagnostics.
19. Add native artifact proof rows for direct dispatch, role-object dispatch, and duplicate symbol counts.
20. Add expected-failure fixture runner with diagnostic category and primary-span matching.

## qtlc1 Rule

qtlc1 must use the advanced surface role model directly. Do not replace the
design with fallback free functions or weak wrapper methods. If qtlc1 needs an
advanced operation, qtlc0 must become capable enough to compile it.
