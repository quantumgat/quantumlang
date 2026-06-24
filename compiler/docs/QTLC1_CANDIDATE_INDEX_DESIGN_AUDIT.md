# qtlc1 Candidate Index Design Audit

Status: locked by `POST-QTLC1-CANDIDATE-INDEX-DESIGN-AUDIT-013B0`.

This audit defines the centralized candidate collection contract required
before `POST-QTLC1-REAL-TYPECHECK-OVERLOAD-GENERIC-013B`.

Candidate collection is a compiler index problem. It must not select an
overload, infer generics, bind arguments, or rank candidates.

## Audit Scope

The audit covered:

- free functions and lexical callable values;
- inherent methods and `impl view` callable members;
- structural and custom constructors;
- Design-10 role conformance methods;
- concrete and bounded blanket extensions;
- imported and re-exported symbols;
- fields and properties used as callable values;
- builtin and role-based operators;
- scope lookup, symbol lookup, expression resolution, and type registry
  handoff.

## Current Findings

### Lookup indexes collapse overloads

`TypeNameLookupIndex` and `TypeScopeLookupIndex` store one row for each key.
Their `add` methods reject a repeated key. This makes them suitable for unique
bindings, but not overload sets.

`TypeSymbolTable` currently has:

- one-row name lookup;
- one-row qualified-name lookup;
- one-row member-call lookup;
- one-row inferred-generic member lookup;
- one-row member-value lookup.

An overload index must store a bucket of candidate IDs for a key.

### Call lookup selects too early

`symbolValueForCall` scans all symbols and returns the first arity-compatible
function or type. It also keeps a first-symbol fallback when no exact arity
matches.

`symbolValueForMemberCall` first asks a one-row index, then scans every symbol
and returns the first generic-owner match.

This is not valid final overload behavior. Declaration order must never select
a call target.

### Expression resolution commits one symbol

`ExpressionResolver.call` currently resolves:

1. one receiver member;
2. one lexical symbol;
3. one global fallback;
4. hardcoded carrier, collection, communication, or print behavior.

It then writes that symbol directly into `ExpressionResolution`.

The final resolver must receive a `CandidateSet`, then pass it to argument
binding, generic inference, validation, and ranking before committing a
symbol.

### Scope lookup is unique-binding lookup

Lexical locals and generic parameters must remain unique inside one scope.
Callable declarations may form an overload set.

The correct shadowing rule is:

1. walk from the current scope toward its parents;
2. stop at the first scope containing the requested name;
3. return every callable candidate in that nearest scope;
4. do not merge hidden candidates from outer lexical scopes.

Module/import candidate collection is separate and follows module visibility.

### Constructors are not real candidates yet

Current type calls treat any `TypeSymbolKind.Type` as a constructor and do not
represent:

- structural auto-constructor parameters;
- custom constructor overloads;
- defaults and named fields;
- target-typed object literals;
- `This(...)`;
- constructor visibility.

Structural and custom constructors must enter the same candidate index.

### Role and extension facts are incomplete

Locked Design 10 requires independent conformance visibility:

```qn
extend User : pub Account, private Auditable, internal Serializable {
}
```

The current qtlc1 `AstDeclaration`, `DeclarationRecord`, and `TypeSymbol`
surfaces retain only a declaration-level public/private flag. They do not yet
preserve one visibility fact per role conformance.

`013B` must add structured conformance rows before indexing role methods.
Candidate lookup must not approximate Design-10 visibility with a boolean.

Bounded blanket extensions also need structured receiver bounds. Textual
generic substitution is not sufficient for applicability checking.

### Operators bypass unified candidates

Builtin operators currently resolve directly through `BuiltinOperatorFact`
and `ExpressionTypeRule`.

The operator registry remains the canonical syntax/form registry, but operator
type resolution must expose candidates through the same candidate pipeline:

1. builtin primitive/core candidate;
2. concrete operator-role candidate;
3. bounded generic operator-role candidate.

### Full scans remain in hot paths

Current full scans include:

- free-call lookup;
- generic member lookup;
- symbol-by-ID and type-by-ID lookup;
- source-position lookup;
- scope ID, owner scope, parent, and node-scope lookup.

`013B` is responsible for call candidate scans. Other identity/scope scans may
be migrated separately, but the candidate resolver must not introduce new
full-symbol scans.

## Locked Ownership Model

`TypeSymbolTable` remains the canonical owner of `TypeSymbol` rows.

`CandidateIndex` owns:

- candidate metadata rows;
- key-to-candidate-bucket indexes;
- stable slices of `CandidateId`;
- bucket generation and dependency fingerprints.

It references symbols by `SymbolId`; it does not duplicate full symbols.

`TypeScopeTable` remains the owner of lexical scopes and unique local
bindings. Callable bindings expose candidate IDs through the centralized
candidate index.

`ModuleGraph` and export surfaces remain the owners of package/module
visibility. Candidate rows retain the resulting access facts.

## Locked Core Records

The implementation should use focused files under:

```text
qtlang/compiler/typeSystem/resolution/
  candidateSource.qn
  candidateAccess.qn
  candidateId.qn
  candidateKey.qn
  candidateEntry.qn
  candidateBucket.qn
  candidateSet.qn
  candidateIndex.qn
  candidateIndexBuilder.qn
```

Required conceptual records:

```qn
pub enum CandidateSource {
  LexicalCallable
  FreeFunction
  InherentMethod
  ViewMethod
  StructuralConstructor
  CustomConstructor
  RoleMethod
  ConcreteExtension
  BlanketExtension
  CallableValue
  BuiltinOperator
  OperatorRole
}

pub enum CandidateAccessKind {
  Public
  Internal
  Protected
  Private
}

pub CandidateAccess {
  kind: CandidateAccessKind
  packageId: PackageId
  moduleId: ModuleId
  ownerType: TypeId
}

pub CandidateEntry {
  id: CandidateId
  symbol: SymbolId
  callableType: TypeId
  ownerType: TypeId
  source: CandidateSource
  access: CandidateAccess
  declarationSpan: SourceSpan
  generic: bool = false
  variadic: bool = false
  defaultedParameterCount: usize = 0
}

pub CandidateSet {
  key: CandidateKey
  candidates: Slice<CandidateId>
  lexicalDepth: usize = 0
  deterministic: bool = true
}
```

`CandidateSet` contains unranked candidates. It has no `selected` field.

## Locked Index Keys

Candidate keys must use stable IDs where semantic identity is known.

```text
FreeCallKey
  package/module visibility context + NameId

LexicalCallKey
  ScopeId + NameId

MemberCallKey
  canonical owner TypeId + NameId

ConstructorKey
  constructed TypeId

ExtensionKey
  canonical receiver TypeId/base TypeId + NameId

OperatorKey
  canonical operator ID + operator form
```

Arity is a bucket filter, not the primary semantic key. Defaults, variadics,
named arguments, and inferred generics mean exact arity cannot identify the
candidate set.

The index may store secondary min/max-arity metadata to reject impossible
buckets cheaply.

String type names may be retained for diagnostics, but they are not final
semantic keys.

## Locked Candidate Collection Order

Collection follows source categories without choosing a winner:

```text
1. nearest lexical callable bucket
2. exact inherent member/custom constructor bucket
3. structural auto-constructor bucket
4. visible role implementation bucket
5. visible concrete extension bucket
6. applicable bounded blanket extension bucket
7. builtin/operator fallback bucket
```

The source category is stored on every candidate. Ranking occurs later.

If a nearest lexical scope contains the name, outer lexical scopes are
shadowed. This does not collapse multiple overloads declared in that nearest
scope.

## Locked Visibility Rules

Candidate collection applies visibility before argument ranking.

Design-10 role visibility is mandatory:

- `pub`: visible through exported package/module boundaries;
- `internal`: visible inside the declaring package;
- `protected`: visible to the target, its extensions, and valid derived or extension contexts;
- `private`: visible only inside the owning type and its extension blocks;
- omitted role visibility: `private`.

Candidates must retain package, module, and owning-type identity so visibility
is checked structurally.

An inaccessible candidate may be retained in a rejected-candidate diagnostic
bucket, but it must never be ranked as viable.

Imports and re-exports must not erase `internal` or `private` access facts.

## Locked Constructor Rules

Candidate index construction synthesizes one structural constructor candidate
for each constructible data type.

Its parameters come from fields:

```text
field without default -> required
field with default    -> optional
field order           -> positional order
field name            -> named argument key
```

Custom constructors are indexed separately under the same `ConstructorKey`.

The index does not bind constructor arguments or evaluate defaults.

## Locked Generic Boundary

Candidate collection does not specialize `TypeSymbol` through string
replacement.

It stores:

- generic parameter IDs;
- receiver pattern type;
- callable type;
- bound IDs;
- explicit owner/method generic ownership.

Structural generic inference belongs to `013D` and consumes the unmodified
candidate rows.

Generic base-name indexing may be used only to collect a broad bucket. It must
not claim applicability.

## Locked Invalidation Model

Candidate indexes are derived products.

Each bucket depends on:

- declaration stable hash;
- callable parameter/generic shape hash;
- owner type identity;
- Design-10 conformance visibility;
- module export/import visibility;
- extension bounds;
- operator registry identity.

A changed declaration invalidates only its affected candidate buckets plus
dependent call-resolution products.

Backend-only changes do not invalidate candidate indexes.

Bucket ordering is stable by `CandidateId` only for deterministic storage and
diagnostics. Stable ordering never selects a winner.

## Complexity Contract

Target complexity:

```text
index construction
  O(number of callable declarations)

candidate lookup
  average O(1) key lookup + O(k) returned candidates

lexical lookup
  O(scope depth) + O(k)

argument binding/ranking
  later phase, O(k * parameter work)
```

Forbidden:

- full `TypeSymbolTable.symbolValues` scan per call;
- first-declaration fallback;
- runtime named-argument maps;
- reflection;
- dynamic dispatch during compiler lookup;
- copying full `TypeSymbol` rows into every bucket.

## Integration Contract

`TypeEnvironment` construction order becomes:

```text
1. TypeSymbolTable
2. TypeScopeTable
3. CandidateIndex
4. TypeRegistry / expression typing
5. diagnostics
```

`ExpressionResolver` must stop calling:

- `symbolValueForCall`;
- `symbolValueForMemberCall`;
- direct first-symbol fallbacks.

It instead builds a call site and asks `CandidateIndex` for a
`CandidateSet`.

`013B` ends after candidate collection. It must not implement argument
binding, generic inference, conversions, ranking, or HIR call commitment.

## Required `013B` Proofs

- two free-function overloads remain in one bucket;
- two member overloads remain in one bucket;
- nearest lexical overload set shadows outer scopes;
- structural and custom constructors coexist;
- Design-10 public role methods are collected cross-package;
- internal role methods are collected only in-package;
- private role methods are collected only in the owner boundary;
- concrete and bounded blanket extensions occupy distinct candidates;
- callable local/field candidates are represented;
- builtin and role-based operator candidates coexist;
- named/default/variadic shapes are retained but not bound;
- candidate order changes do not change resolution results later;
- candidate lookup performs no full-symbol scan.

## Implementation Sequence

`POST-QTLC1-REAL-TYPECHECK-OVERLOAD-GENERIC-013B` must proceed in this order:

1. add structured Design-10 conformance/access rows to syntax, frontend, and
   symbols;
2. add candidate IDs, sources, access facts, keys, entries, buckets, and sets;
3. build free, lexical, member, constructor, extension, and operator indexes;
4. integrate `CandidateIndex` into `TypeEnvironment`;
5. switch `ExpressionResolver` to unranked candidate sets;
6. retain compatibility APIs only as thin single-candidate views for
   unaffected code;
7. add candidate collection and complexity proofs;
8. remove full-scan call/member fallback paths.

The next phase after this is `013C`: positional, named, defaulted, and
variadic argument binding.
