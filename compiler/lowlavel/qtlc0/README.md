# qtlc0 Explicit Ownership Symbols

Status: final low-level design target. qtlc0 implementation migration is
pending.

This package owns the low-level qtlc0 proof for explicit ownership symbols.
The reference, pointer, and move proofs now use the final locked QuantumLang
syntax. This is intentionally ahead of the current qtlc0 parser in places so
the compiler can be fixed toward one clean target.

## Locked Source Rule

```text
value        read borrow/default for non-copy objects
const &value explicit read-only reference
&value       explicit writable/exclusive reference
^value       explicit move/consume
*T           writable pointer target
const *T     read-only pointer target
...value     spread/variadic, already separate and not part of this command
```

Normal function calls must not silently move non-copy aggregates. Ownership
transfer must be visible at the call site with `^`.

## Safety Rule

Safe QuantumLang code should make double-free impossible by construction.
Values and smart owners drop through compiler/runtime lifetime rules. Raw
pointers and references are non-owning views, so safe code cannot manually
destroy through them.

```qn
free(pointer)    // reject for *T
delete(pointer)  // reject for *T
free(owner)      // reject in safe code
delete(owner)    // reject in safe code
```

Pointer target access requires a compiler-proven `pointer.valid` scope. Owners
drop at scope end, at shared-owner last release, at arena release, or during
deterministic global shutdown depending on owner kind.

## Proof Gates

```text
qtlc/build/compiler/driver/qtlc check qtlang/lowlavel/qtlc0
qtlc/build/compiler/driver/qtlc build -j8 qtlang/lowlavel/qtlc0
qtlang/lowlavel/qtlc0/build/debug/qtlc0_explicit_ownership_symbols
```

Target qtlc0 status after migration:

```text
check: passes
build: passes
run: exits 0 and prints qtlc0 explicit ownership symbol proof: ok
rule: normal value calls read-borrow non-copy aggregates by default; only
      ^value consumes ownership
borrow conflicts: explicit ^move, assignment, and overlapping & writable
                  borrow are rejected while a matching borrow is live
pointer gates: &value is target-directed; pointer.valid proves target access
const gate: const *T permits reads and rejects writes
literal matrix: value, const &value, &value, and ^value execute across generic
                nested custom aggregates
```

`src/syntaxIntentProof.qn` uses the real locked syntax:

```qn
const &value
&value
^value
*T
const *T
```

`src/constAuthorityMatrixProof.qn` locks the full binding-const and
target-const matrix:

```qn
let r: &OwnerBox = &box
const r: &OwnerBox = &box

let r: const &OwnerBox = const &box
const r: const &OwnerBox = const &box

let p: *OwnerBox = &box
const p: *OwnerBox = &box

let p: const *OwnerBox = const *box
const p: const *OwnerBox = const *box
```

Binding `const` means the local name cannot be reassigned. Target `const`
means the referenced or pointed-to object is read-only.

Rejected direct assignments:

```qn
let p: const *OwnerBox = &box
let p: const *OwnerBox = const &box
let r: const &OwnerBox = &box
let p: *OwnerBox = const *box
let r: &OwnerBox = const &box
```

These reject because they either hide a reference-to-pointer conversion or
upgrade read-only authority to writable authority. Future explicit conversions
may use named APIs such as `rawPointer()`.

Do not weaken these proofs by extracting scalar fields first. The point of this
package is to prove that normal `value` calls become read-borrow/default and
that only `^value` consumes ownership.

Implemented negative fixture coverage:

```text
use after ^move / move while borrowed
write while read-only referenced
read-only reference while writable referenced
```

The registered generic/custom negative fixtures are:

```text
ownership_literal_use_after_move_bad.qn
ownership_literal_move_while_borrowed_bad.qn
ownership_literal_readonly_write_conflict_bad.qn
ownership_literal_writable_reference_conflict_bad.qn
```

Implemented pointer negative fixture coverage:

```text
pointer field access outside pointer.valid
write through const *T
```

The proof executable also prints address tokens derived from a normal reference
and from the raw pointer wrapper, followed by real native address formatting:

```text
qtlc0 ref address proof: 7
qtlc0 raw pointer address proof: 7
address: 0x...
ptr address: 0x...
```

Locked interpolation rule:

```qn
let value: i64 = 42
let ptr: const *i64 = const *value

print("address: {&value}\n")
print("ptr address: {ptr}\n")
```

`&value` and a pointer value format as hexadecimal native addresses.

Object targets use direct fields after one validity proof:

```qn
let pointer: *User = &user
if pointer.valid {
  pointer.name = "Ali"
  print(pointer.name)
}

let readonly: const *User = const *user
if readonly.valid {
  print(readonly.name)
}
```

`*T` is writable and `const *T` is read-only. There is no public `raw` keyword
and no unsafe block in this model.

Pointer authority is centralized:

- `let pointer: *T = &value` creates an exclusive authority loan.
- `let pointer: const *T = const *value` creates a shared authority loan.
- `pointer.valid` proves owner lifetime, provenance, bounds, alignment, and
  non-null state for the guarded CFG path.
- `if pointer.valid && otherCondition` and `while pointer.valid` narrow the
  pointer only inside the true/body region.
- Mutating fields or methods through `const *T` are compile errors.
- Legacy unary `*pointer` dereference is restricted to compiler-core/low-level
  implementation code.
- Compiler-derived local addresses are statically non-null, so native lowering
  can remove the redundant runtime validity test.

## Owner And Lifetime Matrix

`src/smartOwnerGlobalPointerProof.qn` locks the owner categories:

```text
Value/local owner:
  let user: User = { ... }
  Owns storage in current scope.
  Dropped at scope end.
  Pointers/references into it cannot escape scope.

Unique<T>:
  Unique smart pointer owner.
  Owns one allocation or native object identity.
  Dropped automatically.
  Movable with ^.
  Not copyable.
  No safe release to raw pointer.

Shared<T>:
  Shared smart owner.
  Reference-counted or compiler-managed shared ownership.
  Copy/clone policy must be explicit.
  Dropped when last owner ends.

Weak<T>:
  Non-owning link to Shared<T>.
  Must upgrade/check before use.
  Does not keep object alive.

Pin<T>:
  Stable-address owner.
  Object cannot move after pin.
  Good for async, intrusive structures, native ABI, and self references.

Arena<T>:
  Region owner.
  Many values allocated inside arena.
  Everything dropped/released when arena ends.
  Individual raw pointers are valid only while arena is alive.

Global<T>:
  Global smart owner.
  Program-lifetime or deterministic global lifetime.
  Dropped at shutdown in compiler-known order.

Permanent<T>:
  Intentional process-lifetime owner.
  No normal drop.
  Only for runtime/compiler/system cases where the leak is explicit design.

Handle<T>:
  External resource owner: file, socket, GPU handle, OS handle.
  Compiler drops/closes at scope end.
  close() can be idempotent.
```

Smart owner operations:

```text
Common:
  valid
  owns
  ref()
  cref()
  ptr()
  cptr()
  address
  generation
  take()
  replace(value)
  swap(other)
  ^owner

Unique<T>:
  Unique<T>(value)
  unique.valid
  unique.empty
  unique.value
  unique.ref()
  unique.cref()
  unique.ptr()
  unique.cptr()
  unique.address
  unique.take()
  unique.replace(value)
  unique.reset(value)
  unique.clear()
  unique.swap(other)
  unique.intoShared()
  unique.intoPin()
  unique.intoPermanent()
  ^unique

Shared<T>:
  Shared<T>(value)
  shared.alias()
  shared.weak()
  shared.ref()
  shared.cref()
  shared.ptr()
  shared.cptr()
  shared.strongCount
  shared.weakCount
  shared.tryUnique()
  shared.makeUnique()

Weak<T>:
  weak.valid
  weak.expired
  weak.upgrade()
  weak.canUpgrade
  weak.generation

Pin<T>:
  Pin<T>(value)
  pin.ref()
  pin.cref()
  pin.ptr()
  pin.cptr()
  pin.stableAddress
  pin.project(field)
  pin.unpin()

Arena<T>:
  Arena<T>(capacity)
  arena.allocate(value)
  arena.allocateConst(value)
  arena.allocateMany(values)
  arena.mark()
  arena.rewind(mark)
  arena.reset()
  arena.seal()
  arena.contains(pointer)
  arena.owns(pointer)
  arena.count
  arena.capacity
  arena.remaining
  arena.generation
  arena.asSlice()

Global<T>:
  Global<T>(value)
  globalOwner.ref()
  globalOwner.cref()
  globalOwner.ptr()
  globalOwner.cptr()
  globalOwner.shutdownOrder
  globalOwner.dependsOn(other)

Permanent<T>:
  Permanent<T>(value)
  permanent.ref()
  permanent.cref()
  permanent.ptr()
  permanent.cptr()
  permanent.processLifetime

Handle<T>:
  Handle<T>(native)
  handle.valid
  handle.open
  handle.close()
  handle.closeOnce()
  handle.takeNative()
  handle.native()
  handle.flush()
  handle.sync()
```

Local values cover ordinary scoped ownership. `Unique<T>` is the unique smart
pointer owner with stable allocation identity. It is non-copyable and movable
only with `^`.

Raw/reference rules:

```text
&T              writable reference, non-owning
const &T        read-only reference, non-owning
*T              writable raw pointer, non-owning
const *T        read-only raw pointer, non-owning
```

No manual `free` or `delete` in safe code:

```qn
free(p)        // reject for *T
delete(p)      // reject for *T
free(owner)    // reject in safe code
delete(owner)  // reject in safe code
```

Global pointers require a global/static/permanent/arena owner with compatible
lifetime:

```qn
global config: User = { name: "Config" }
global configPtr: const *User = const *config
```

Allowed because `config` is global, so `configPtr` cannot outlive it.

Writable global pointers require exclusive global authority:

```qn
global state: User = { name: "A" }
global statePtr: *User = &state
```

qtlc must reject conflicting writable global aliases unless the owner is
protected by `Atomic`, `Mutex`, `Cell`, or a stronger synchronization type.

Rejected global escape:

```qn
global leaked: *User

pub makeLeakedPointer() {
  let user: User = { name: "A" }
  leaked = &user
}
```

The local owner cannot escape to a global pointer. Shutdown is deterministic:
global owners drop in dependency order, global pointers are invalidated before
or with their owner, and `Permanent<T>` skips normal shutdown drop by explicit
design.

## Deep OOP Pointer Surface

`POST-QTLC0-POINTER-DEEP-OOP-SURFACE-PROOF-007` combines:

- writable and `const *` generic object pointers;
- deep object fields and nested `impl view` chains;
- generic roles and role dispatch;
- bounded blanket extensions;
- `context` and `extract` methods;
- extensions imported from another module;
- direct native field loads/stores without reflection or dynamic dispatch.

Native proof evidence:

```text
native_aggregate_lowering=direct field-access
native_aggregate_lowering=direct field-mutation
native_inline_sroa=direct
```

The generated proof path contains no indirect call, reflection lookup, vtable
dispatch, named-argument map, or heap allocation. A one-file proof edit rebuilds
one native object and links against immutable cached objects selected by source
identity.

## Ownership Literal Matrix

`POST-QTLC0-OWNERSHIP-LITERAL-FULL-MATRIX-PROOF-009` replaces the old
placeholder comments with executable source:

```qn
defaultRead(value)
sharedRead(const &value)
mutate(&value)
consume(^value)
```

The proof covers generic envelopes, nested custom payloads, deep field access,
computed views, mutation through a writable reference, and explicit ownership
transfer. The executable exits `0`; all four registered negative diagnostics
pass; the immediate no-change build reports `outputs up to date` in `00:00`.

## Concrete Generic Aggregate Layout

`POST-QTLC0-GENERIC-AGGREGATE-CONCRETE-LAYOUT-010` proves that generic native
layout and ABI decisions use the concrete payload:

```qn
let text: LayoutCell<string> = { value: "quantum" }
let pair: LayoutCell<LayoutPair> = {
  value: { left: 11, right: 29 }
}
let nested: LayoutEnvelope<string> = {
  cell: { value: "nested" }
}
```

The native backend recursively substitutes generic field types, recomputes
offsets, selects exact monomorphs, preserves two-word text returns and
aggregate sret returns, and forwards nested aggregate receivers by address.
The proof runs together with the deep pointer/OOP suite and exits `0`, with no
allocation, reflection, dynamic dispatch, C/C++, or LLVM fallback.
