# qtlc0 Ownership, References, And Pointer Targets

Status: locked and implemented for the qtlc0 seed surface.

## Locked Rule

```qn
value             // read borrow/default for non-copy values
&value            // explicit shared reference/address
&!value           // explicit mutable reference
^value            // explicit ownership move

let read: &T = &value
let write: &!T = &!value
let pointer: *T = &value
let readonly: *const T = &value
let inferred = &value  // inferred as &T
```

There is no public `raw` keyword. The target type determines whether `&value`
is a safe reference, mutable reference, mutable pointer, or read-only pointer.

Internal canonical pointer names remain `*mut T` and `*const T` so native ABI
and cache metadata have one stable representation. Source `*T` normalizes to
internal `*mut T`.

## Pointer Validity

Pointer target access requires one compiler-recognized validity scope:

```qn
let pointer: *User = &user

if pointer.valid {
  pointer.name = "Ali"
  pointer.active = true
  print(pointer.name)
}
```

`pointer.valid` proves:

- non-null address
- live provenance
- target alignment
- target bounds
- compatible target type

Native lowering uses a direct address check and direct offset loads/stores. It
does not allocate, reflect, dispatch dynamically, or build a runtime field map.

## Const Target

```qn
let pointer: *const User = &user

if pointer.valid {
  print(pointer.name)
  pointer.name = "Ali" // compile error
}
```

`const` applies to the pointed target. It allows reads and rejects writes.

## Address Formatting

```qn
let value: i64 = 42
let pointer: *const i64 = &value

print("address: {&value}\n")
print("pointer: {pointer}\n")
```

Both interpolations render the real native address as hexadecimal.

## Safety Diagnostics

The compiler rejects:

- field access outside `if pointer.valid`
- field mutation outside `if pointer.valid`
- mutation through `*const T`
- use after `^value`
- moving while borrowed
- mutable/shared borrow conflicts
- returning an address derived from a dead local

Normal `value` use does not silently move non-copy data. Only `^value`
consumes ownership.

## Implementation Map

```text
POST-QTLC0-POINTER-TARGET-SURFACE-001
  accept *T and normalize it to internal *mut T
  keep *const T as the read-only target form

POST-QTLC0-TARGET-DIRECTED-ADDRESS-002
  infer &value as &T
  coerce &value to &!T, *T, or *const T from the destination type
  remove public &raw syntax

POST-QTLC0-POINTER-VALIDITY-NARROWING-003
  type pointer.valid as bool
  narrow the first if-body to a validated pointer scope
  reject target access outside that scope

POST-QTLC0-CONST-POINTER-WRITE-GATE-004
  permit reads through *const T
  reject field writes through *const T

POST-QTLC0-NATIVE-POINTER-HOT-PATH-005
  lower pointer.valid to a direct non-null check
  lower pointer fields to direct native offset loads/stores
  format &value and pointer values as native hexadecimal addresses
```

## Proofs

Positive package:

```text
qtlang/lowlavel/qtlc0
```

Negative fixtures:

```text
qtlc/tests/fixtures/pointer_field_requires_valid_bad.qn
qtlc/tests/fixtures/const_pointer_write_bad.qn
qtlc/tests/fixtures/raw_address_keyword_removed_bad.qn
```

Required gates:

```text
qtlc check qtlang/lowlavel/qtlc0
qtlc build -j8 qtlang/lowlavel/qtlc0
qtlang/lowlavel/qtlc0/build/debug/qtlc0_explicit_ownership_symbols
```

After this surface, qtlc1 resumes at:

```text
POST-QTLC1-FRONTEND-DECLARATION-ARENA-003
```
## POST-QTLC0-POINTER-PROVENANCE-AUTHORITY-006

- Central pointer facts own owner, lifetime, mutability, authority, bounds,
  alignment, nullability, and provenance generation.
- `*T` requires exclusive authority; `*const T` carries shared authority.
- Owner move, drop, or assignment invalidates dependent pointer provenance.
- `pointer.valid` narrows direct, conjunctive, nested, and loop CFG paths.
- Const pointers reject field mutation and methods declared `mut`.
- Proven local-address pointers fold validity checks in native code.
- Legacy unary pointer dereference is reserved for compiler-core/low-level code.
- Positive runtime proof: `qtlang/lowlavel/qtlc0`.
- Negative proofs cover authority conflicts, owner mutation, const method calls,
  unguarded access, and public legacy dereference.

## POST-QTLC0-POINTER-AUTHORITY-SSA-007

- Mutable raw pointers are exclusive authority tokens and cannot be copied.
- `^pointer` explicitly transfers mutable pointer authority.
- Read-only `*const T` aliases preserve owner and lifetime provenance.
- Local-derived raw pointers cannot cross a call without a typed `noescape`
  contract.
- Pointer validity uses `Unknown`, `Proven`, and `Invalid`, not a boolean fact.
- Assigning a pointer invalidates its current `pointer.valid` CFG proof.
- Native lowering does not fold `pointer.valid` from initializer shape alone.
- Negative proofs cover mutable aliases, call escape, and reassignment after
  validity narrowing.
