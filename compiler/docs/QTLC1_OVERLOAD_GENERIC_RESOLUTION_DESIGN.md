# qtlc1 Overload And Generic Resolution Design

Status: locked by `POST-QTLC1-OVERLOAD-GENERIC-DESIGN-AUDIT-012`

This page defines the resolver contract that
`POST-QTLC1-REAL-TYPECHECK-OVERLOAD-GENERIC-013` must implement.

Candidate collection details are separately locked by
[QTLC1_CANDIDATE_INDEX_DESIGN_AUDIT.md](QTLC1_CANDIDATE_INDEX_DESIGN_AUDIT.md).

The resolver is compile-time infrastructure. It must not use runtime
reflection, a named-argument map, dynamic dispatch, source declaration order,
or repeated full-symbol scans.

## Audit Result

The current qtlc1 type checker has useful foundations:

- stable `SymbolId`, `TypeId`, `ScopeId`, and declaration identities;
- indexed name, qualified-name, scope, and exact-owner member lookup;
- constructor, function, member, field, and property symbols;
- parser-provided call and generic arity;
- central operator and diagnostic registries;
- owner-generic substitution for one simple generic argument.

The current model is not sufficient for real overload or generic resolution:

- calls preserve counts, but not structured argument rows;
- parameters and generic parameters are stored mainly as strings;
- candidate lookup returns one symbol instead of a candidate set;
- matching is primarily name plus exact arity;
- generic specialization uses textual single-argument substitution;
- default, named, variadic, role-bound, extension, and callable-value matching
  are not represented;
- conversion cost and rejection reasons are not represented;
- constructor auto-fields and custom constructors are not unified as
  candidates;
- equal candidates can only be handled correctly after deterministic ranking
  and ambiguity reporting exist.

`...-013` must replace these shortcuts rather than extend them.

## Resolver Pipeline

Resolution has seven ordered stages:

```text
1. build CallSite
2. collect indexed candidates
3. bind arguments
4. infer and validate generic substitutions
5. validate types, bounds, visibility, capabilities, and ownership
6. rank viable candidates
7. commit one ResolutionProduct or emit structured diagnostics
```

Candidate collection may be broad and cheap. Candidate validation must be
structural and exact. No stage may select the first declaration encountered.

## Required Data Contracts

`...-013` should add structured records equivalent to:

```qn
pub CallArgument {
  expression: NodeId
  name: NameId?
  typeId: TypeId
  position: usize
  spread: bool = false
}

pub CallableParameter {
  name: NameId
  typeId: TypeId
  position: usize
  defaultExpression: NodeId?
  variadic: bool = false
}

pub GenericParameter {
  name: NameId
  typeId: TypeId
  position: usize
  bounds: Slice<RoleBoundId> = []
  defaultType: TypeId?
  variadic: bool = false
}

pub GenericSubstitution {
  parameter: TypeId
  argument: TypeId
  source: GenericInferenceSource
}

pub ArgumentBinding {
  argument: NodeId?
  parameter: usize
  conversion: ConversionFact
  defaulted: bool = false
  variadic: bool = false
}

pub CallCandidate {
  symbol: SymbolId
  callableType: TypeId
  source: CandidateSource
  receiverDistance: usize = 0
  bindings: Slice<ArgumentBinding> = []
  substitutions: Slice<GenericSubstitution> = []
  cost: CandidateCost = {}
  rejection: CandidateRejection = {}
}

pub CallResolution {
  selectedSymbol: SymbolId
  instantiatedType: TypeId
  bindings: Slice<ArgumentBinding> = []
  substitutions: Slice<GenericSubstitution> = []
  conversions: Slice<ConversionFact> = []
  dispatch: DispatchKind
  valid: bool = false
}
```

The final implementation may split these records into focused modules, but it
must preserve this information. Counts and strings are report views, not the
source of truth.

## Candidate Sources

All callable forms enter one resolver:

```text
free function
callable local or field: (A...) -> R
structural auto-constructor
custom constructor
This(...) constructor call
inherent impl method
impl view method when called with ()
role implementation method
imported extension method
bounded blanket extension
builtin operator candidate
user operator-role candidate
```

Candidate source precedence narrows lookup but does not silently hide an
equally valid conflict:

```text
1. lexical callable binding
2. exact inherent member or custom constructor
3. structural auto-constructor
4. explicit role implementation in scope
5. imported concrete extension
6. bounded blanket extension
7. builtin fallback where the operator/type contract permits it
```

An exact concrete candidate ranks ahead of a generic blanket candidate.
Candidates equal after the full cost comparison are ambiguous.

## Candidate Index Boundary

`POST-QTLC1-CANDIDATE-INDEX-DESIGN-AUDIT-013B0` found that current qtlc1
name/scope indexes store one row per key, free/member calls select one symbol
too early, generic-member fallback scans the full symbol table, constructors
are not represented as real candidates, and Design-10 per-role visibility is
not yet preserved by qtlc1 declaration records.

`013B` must therefore implement the centralized candidate-index contract
before argument binding:

- `TypeSymbolTable` owns symbols;
- `CandidateIndex` owns key-to-candidate buckets and access/source metadata;
- `CandidateSet` is unranked and has no selected symbol;
- arity is a secondary filter, not the primary lookup key;
- role conformance visibility is structural
  `pub/internal/protected/private` metadata;
- candidate lookup is average `O(1) + O(k)` and performs no full-symbol scan.

Implementation status:

- `013B` now provides the centralized unranked candidate boundary and exact
  role-method visibility indexing;
- `protected` is a distinct access kind alongside `pub`, `internal`, and
  `private`;
- argument binding, generic inference, conversion validation, cost ranking,
  and ambiguity diagnostics remain `013C`.

See the candidate-index audit for exact keys, ownership, invalidation,
complexity, integration order, and proof requirements.

## Constructor Contract

Structural and custom constructors share the same candidate pipeline.

Structural auto-constructor parameters derive from fields:

```text
field without default -> required parameter
field with default    -> optional parameter
field declaration order defines positional order
field name defines named-argument identity
```

Locked rules:

- positional arguments come before named arguments;
- a named argument may skip optional fields;
- unknown or duplicate names are errors;
- every required field must be bound;
- default expressions are evaluated only for omitted fields;
- custom constructors remain candidates and may validate or normalize;
- `This()` recursively calling the same empty constructor is rejected;
- target-typed object literals use the same binding engine;
- lowering uses direct aggregate stores, never a runtime argument map.

## Argument Binding

Argument binding occurs before type ranking.

The binder must:

1. bind positional arguments by parameter position;
2. bind named arguments by stable `NameId`;
3. reject positional arguments after the first named argument;
4. reject duplicate and unknown names;
5. fill omitted defaulted parameters;
6. collect a variadic tail only for a declared variadic parameter;
7. preserve source argument order for evaluation;
8. preserve parameter order for lowering.

Arity indexing may eliminate impossible candidates, but arity alone never
selects a candidate.

## Generic Inference

Generic inference is structural over `TypeId` graphs, not string replacement.

It must support:

- multiple type parameters;
- nested carriers such as `Result<Vec<T>, E>`;
- owner and method generic parameters together;
- explicit, inferred, and defaulted generic arguments;
- repeated occurrences that must infer the same type;
- callable, tuple, collection, quantum, pointer, reference, and custom types;
- variadic generic packs when the syntax declares them;
- role/type bounds and bounded blanket extensions.

Inference sources are ordered:

```text
explicit generic argument
receiver type
typed call argument
expected return/target type
generic default
```

Later sources may complete an unknown substitution but may not overwrite a
conflicting earlier substitution.

Expected return type may disambiguate otherwise viable overloads. It must not
make an argument-incompatible candidate viable.

## Bounds And Extension Visibility

A generic candidate is viable only when every bound is proven in the current
module and capability environment.

Required checks:

- role implementation exists for the concrete substitution;
- bounded blanket extensions use proven bounds;
- locked Design-10 `pub/internal/private` conformance and extension visibility
  is respected;
- foreign extensions do not gain private field access;
- conflicting role or extension members require explicit selection;
- unsafe or capability-gated candidates require explicit authority;
- no dynamic role-object dispatch is introduced into qtlc1.

qtlc1 uses static dispatch for these surfaces. Full runtime `any Role`
dispatch remains a later-stage capability.

## Conversion Model

Every argument match produces a typed `ConversionFact`.

Implicitly viable conversions:

```text
exact type identity
generic identity after substitution
safe numeric widening
literal-to-target specialization when the literal fits
reference/reborrow adjustment allowed by ownership rules
optional/result carrier construction explicitly permitted by language rules
```

Not implicit:

```text
lossy numeric narrowing
signed/unsigned reinterpretation
unsafe pointer conversion
string parsing
allocation-backed collection conversion
user conversion with observable side effects
```

These require explicit syntax or an explicit method.

## Deterministic Ranking

Ranking uses a lexicographic `CandidateCost`; lower is better:

```text
1. rejection state
2. candidate source rank
3. receiver match cost
4. total argument conversion cost
5. worst individual conversion cost
6. generic inference cost
7. defaulted parameter count
8. variadic use count
9. capability/unsafe adaptation cost
10. specialization distance
```

Conversion cost order:

```text
0 exact type
1 exact type after generic substitution
2 literal target specialization
3 safe widening or read-only reborrow
4 permitted carrier/context adaptation
```

Lossy or unsafe implicit conversions are rejected, not assigned a high cost.

Stable symbol IDs may order diagnostic output only. They must never choose a
winner. Equal viable cost means `AmbiguousOverload`.

## Operators

The central operator registry defines syntax, form, precedence, arity,
mutation, and builtin eligibility. Operator type resolution uses the same
candidate, generic, conversion, and ranking contracts as calls.

Resolution order:

```text
builtin exact primitive/core rule
concrete operator role implementation
bounded generic operator role
diagnostic
```

Assignment operators additionally require a writable target. Prefix, postfix,
reference, move, dereference, propagation, optional-type, call, index, path,
and spread forms remain distinct contextual operator forms.

## Callable Values

A value typed `(A...) -> R` is directly callable.

- its parameter and return types are already structural `TypeId` records;
- it is not searched as a declaration overload set;
- generic callable values carry their own generic parameter environment;
- calling an `Option<callable>` requires explicit Option handling;
- invocation remains static unless a later explicit dynamic callable type is
  introduced.

## Lookup And Cache Contract

The resolver must not scan all symbols per call.

Required indexes:

```text
(scope, name, call-shape) -> Slice<SymbolId>
(owner TypeId, name, call-shape) -> Slice<SymbolId>
(role TypeId, receiver TypeId, name) -> Slice<SymbolId>
(operator id, operand type family) -> Slice<SymbolId>
```

`call-shape` contains cheap filters only:

```text
positional count
named argument name set/hash
explicit generic count
receiver present
constructor/member/operator form
```

Resolution cache key:

```text
call-site NodeId
scope/environment generation
receiver TypeId
argument TypeIds and names
explicit generic TypeIds
expected TypeId
capability/ownership environment
target and profile identity
```

The cached product stores the selected symbol, substitutions, bindings,
conversions, and diagnostics. A body-only edit invalidates only affected call
sites; a public signature edit invalidates users of that signature.

## Diagnostics Contract

`...-013` must extend `TypeDiagnosticCode` and `DiagnosticRegistry` with
specific structured failures:

```text
NoMatchingOverload
AmbiguousOverload
WrongArgumentCount
UnknownNamedArgument
DuplicateNamedArgument
PositionalAfterNamed
MissingRequiredArgument
GenericArityMismatch
GenericInferenceFailed
GenericInferenceConflict
GenericBoundUnsatisfied
CandidateNotVisible
CapabilityRequired
RecursiveConstructor
InvalidImplicitConversion
```

Diagnostics must include the call span, rejected candidate summaries, the
best mismatch reason, and useful notes for equal viable candidates. They must
not expose unstable internal row numbers.

## Typed AST And HIR Handoff

A resolved call cannot be represented only by result type and arity.

`TypedNodeFact` and HIR must receive:

```text
selected SymbolId
instantiated callable TypeId
resolved result TypeId
receiver adjustment
argument-to-parameter bindings
generic substitutions
implicit conversion facts
default argument expressions
dispatch kind
operator identity when applicable
```

MIR lowering consumes this committed plan and performs no overload search.

## Required Proof Matrix

`...-013` is not complete until executable positive and negative proofs cover:

- exact free-function overloads;
- named, positional, mixed, defaulted, and invalid argument binding;
- structural and custom constructor overloads;
- zero, one, multiple, nested, and inferred generic parameters;
- conflicting generic inference;
- role bounds and bounded blanket extensions;
- inherent versus concrete extension versus blanket extension ranking;
- cross-module visibility and explicit role conflict selection;
- callable values;
- builtin and role-based operators;
- expected-return-type disambiguation;
- ambiguity independent of declaration order;
- no-match diagnostics with candidate notes;
- stable cached resolution across no-change builds;
- one signature change invalidating only dependent call sites;
- native lowering with no reflection, argument map, or dynamic dispatch.

## Implementation Order For 013

```text
013A structured parameter, generic parameter, and call argument rows
013B overload candidate indexes returning candidate slices
013C positional/named/default/variadic argument binder
013D structural TypeId unification and generic substitutions
013E role bounds, extension visibility, and capability viability
013F conversion facts and lexicographic ranking
013G structured overload/generic diagnostics
013H TypedNodeFact and HIR committed call plan
013I resolution query cache and invalidation proof
013J executable, negative, native, and incremental proof matrix
```

Implementation must keep exact indexed lookup as the hot path and must fix
qtlc0 when required Quantum language capability is missing. qtlc1 source must
not be weakened to fit bootstrap shortcuts.
