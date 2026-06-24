# qtlc1 Core Type, Memory, And Operator Scope

Status: scope lock before frontend/type-system implementation

This document locks the qtlc1 compiler-known core surface. qtlc1 must know
these types, literals, memory facts, and operators well enough to compile
qtlc1 into qtlc2. qtlc1 must not implement the runtime or stdlib behind them.

## Bootstrap Sources Checked

The qtlc0 bootstrap currently recognizes primitive and low-level type names in
these places:

```text
qtlc/compiler/tir/src/tir_lowerer.cpp
  ABI classifies scalar, pointer, array, quantum, text/bytes, Option, Result

qtlc/compiler/abi/src/ffi_boundary.cpp
  FFI boundary accepts the broad public primitive/type surface

qtlc/compiler/abi/src/layout_lowerer.cpp
  ABI layout computes size/align for primitives, pointers, arrays, strings,
  bytes, wide numerics, decimal/fixed, complex, and big integers

qtlc/compiler/machine/src/machine_calling_convention.cpp
  return ABI distinguishes register, void, and sret carriers

qtlc/tests/fixtures/types10_canonical_dump_surface.qn
  fixture covers canonical primitive spellings, pointers, arrays, tuples,
  function pointers, fixed, complex, BigInt, BigUint, Text/Bytes views
```

qtlc1 should use this list as bootstrap compatibility input, then normalize to
the final QuantumLang style where the language design is already clear.

## qtlc1 Rule

qtlc1 owns compiler facts for core types:

```text
parse/type identity
name normalization and compatibility aliases
literal typing
operator typing
constant-fold eligibility
layout and ABI facts
target support facts
diagnostics
incremental cache fingerprints
```

qtlc1 does not own implementations:

```text
allocator implementation
Vec/Map/Set storage implementation
BigInt/BigUint arithmetic implementation
decimal/fixed arithmetic implementation
complex math library
string/bytes runtime implementation
smart pointer library
OS memory mapping
SIMD library
crypto library
```

Those implementations move to qtlc2/qtlc3 core/runtime/stdlib/toolchain work.

## Catalog Ownership

qtlc1 must not put every built-in type rule in one large file. The type system
uses separate catalogs:

```text
types/
  Type, TypeId, TypeContext

generic/
  generic parameter facts

inference/
  inference state/facts

checking/
  type checker entry points

effect/
  effects and capability sets

catalog/primitive/primitiveTypeCatalog.qn
  machine/compiler primitive facts only:
  void, bool, char, byte, signed integers, unsigned integers, floats

catalog/core/coreTypeCatalog.qn
  non-primitive core families:
  text/bytes, decimal/fixed/big, complex/math, carriers, collections, quantum

catalog/alias/typeAliasCatalog.qn
  compatibility aliases only:
  int -> i64
  uint -> u64

  Structural normalizations only:
  matrix<N,N,T> -> matrix<N,T>

catalog/layout/typeLayoutCatalog.qn
  ABI/layout facts, backed by PrimitiveTypeFact when the type is primitive

catalog/builtin/builtinTypeCatalog.qn
  small public facade only; no long primitive lists
```

Primitive facts are first-class:

```text
PrimitiveTypeFact {
  name
  family
  bits
  signed
  sizeBytes
  alignBytes
  pointerSized
  abiReturn
  targetGated
  valid
}
```

This keeps `i128`, `u128`, `i256`, and `u256` as real compiler facts rather
than names hidden behind generic boolean checks.

## Primitive Type Surface From qtlc0

### Void, Unknown, And Internal Sentinels

```text
void
Unknown
VarArgs
```

qtlc1 canonical public no-value type is `void`.

New qtlc1 source must use `void` for explicit no-value return types, or omit
the return type for a void block method. It must not use `Unit` or the empty
tuple spelling as no-value style. `Unit` and `()` remain qtlc0 historical
inputs only; qtlc1 does not keep them as aliases.

`Unknown` and `VarArgs` are compiler-internal diagnostic/sentinel types. They
must not become normal user-facing qtlc1 types.

### Boolean

```text
bool
```

qtlc1 canonical public type is `bool`.

`Bool` is not a qtlc1 alias.

### Character And Byte

```text
char
byte
```

qtlc1 canonical model:

```text
char
  Unicode scalar value, 32-bit layout in qtlc0 ABI facts

byte
  canonical raw byte type, same size/layout as u8 but byte-data semantics

u8
  canonical unsigned 8-bit arithmetic integer
```

### Text And Bytes

```text
string
str
bytes
BytesView
BytesMutView
```

qtlc1 canonical model:

```text
string
  owned or compiler-mode text value, runtime-managed ABI carrier

str
  borrowed text view type, real type, not an alias of string

bytes
  owned byte buffer/value, runtime-managed ABI carrier

BytesView
  borrowed read-only byte view, real type, not an alias of bytes

BytesMutView
  borrowed mutable byte view, real type, not an alias of bytes
```

`Text`, `String`, and `Bytes` are not qtlc1 aliases. qtlc1 source uses
`string`, `str`, `bytes`, `BytesView`, and `BytesMutView` as distinct real
types.

qtlc1 may type-check text/bytes values and method lookup when compiler source
needs them. Real Unicode, allocation, formatting, parsing, and byte-buffer
implementations belong to qtlc2/qtlc3.

### Signed Integers

```text
i8
i16
i32
i64
i128
i256
isize
int
```

qtlc1 canonical defaults:

```text
integer literal with no target type -> i32 when small and unambiguous
isize -> target pointer-width signed integer
int -> compatibility alias, not preferred in new qtlc1 source
```

`i128` and `i256` require wide-integer layout and ABI facts. qtlc1 only needs
the facts; arbitrary-precision arithmetic implementation is not qtlc1 scope.

Required qtlc1 signed integer facts:

```text
i8     bits=8    size=1   align=1   sret=false
i16    bits=16   size=2   align=2   sret=false
i32    bits=32   size=4   align=4   sret=false
i64    bits=64   size=8   align=8   sret=false
i128   bits=128  size=16  align=16  sret=true
i256   bits=256  size=32  align=16  sret=true target-gated
isize  bits=target-pointer-width pointer-sized
```

### Unsigned Integers

```text
u8
u16
u32
u64
u128
u256
usize
uint
```

qtlc1 canonical defaults:

```text
usize -> target pointer-width unsigned integer
uint -> compatibility alias, not preferred in new qtlc1 source
```

Required qtlc1 unsigned integer facts:

```text
u8     bits=8    size=1   align=1   sret=false
u16    bits=16   size=2   align=2   sret=false
u32    bits=32   size=4   align=4   sret=false
u64    bits=64   size=8   align=8   sret=false
u128   bits=128  size=16  align=16  sret=true
u256   bits=256  size=32  align=16  sret=true target-gated
usize  bits=target-pointer-width pointer-sized
```

### Floating Point

```text
f16
f32
f64
f128
```

qtlc1 should type-check these names and target support facts. `f16` and
`f128` can be target-gated. Full math library behavior is not qtlc1 scope.

Required qtlc1 float facts:

```text
f16   bits=16   size=2   align=2   sret=true target-gated
f32   bits=32   size=4   align=4   sret=false
f64   bits=64   size=8   align=8   sret=false
f128  bits=128  size=16  align=16  sret=true target-gated
```

### Decimal, Fixed, Big, And Complex Numbers

```text
dec
dec64
dec128
dec256
fixed<T, SCALE>
BigInt
BigUint
complex64
complex128
complex256
complex<T>
rat64
simd<T, N>
vector<N, T>
matrix<N, T>
matrix<R, C, T>
tensor<D1, D2, ..., T>
field<64>
overflow<T>
```

qtlc0 layout and ABI facts recognize this family across ABI, layout, and native
return handling. qtlc1 must carry type identity, layout, target support, and
operator diagnostics for these names.

Canonical qtlc1 math names must stay readable and parameterized. Avoid public
names such as `Vector3F64`, `SimdI64x4`, and `Matrix3x3F64` in new compiler
source. qtlc1 does not keep those old names as aliases.

Structural normalization:

```text
matrix<2,2,f64> -> matrix<2,f64>
matrix<3,3,f64> -> matrix<3,f64>
```

This is not a legacy alias. `matrix<R,C,T>` is valid for rectangular matrices;
when `R == C`, qtlc1 may normalize to `matrix<N,T>`.

qtlc1 must not implement the full math runtime. Decimal/fixed/BigInt/BigUint
and complex arithmetic are qtlc2/qtlc3 core/runtime/stdlib work.

Note:

```text
dec32
```

appears in some qtlc0 diagnostics/tests as a policy surface, but the core ABI
paths checked here center on `dec`, `dec64`, `dec128`, and `dec256`. qtlc1
should keep `dec32` behind an explicit compatibility gate until bootstrap ABI
acceptance is verified.

### Option And Result Carriers

```text
Option<T>
Result<T, E>
T?
none
```

qtlc1 canonical style:

```text
T? == Option<T>
plain T value lifts into T? when the target type is explicit
none
```

qtlc1 must implement carrier typing, pattern/binding checks needed by compiler
source, `?` propagation typing, and ABI/layout facts for cache correctness.

qtlc1 must not implement a full stdlib result/option method library unless
qtlc1 compiler source requires one tiny method and it is compiler-local.

### Function And Callable Types

```text
fn(...)
fn(...) -> T
fn(...) -> void
async fn(...)
qfunc
qfunc(...)
qfunc(...) -> T
qfunc(...) -> void
```

qtlc1 must parse and type-check function pointer/callable shapes needed by
compiler source and quantum facts. Full async runtime behavior is not qtlc1
scope.

### Pointer, Reference, And Slice-Like Types

```text
&T
&mut T
*const T
*mut T
*T
null
NonNull<T>
OptionPtr<T>
[]T
[T; N]
Slice<T>
MutSlice<T>
SliceView<T>
MutSliceView<T>
```

qtlc1 must model:

```text
pointer width and align
mutability
nullability
borrow/reference identity
raw pointer identity
null literal typing for nullable/raw pointer surfaces
fixed array length
slice pointer + length facts
unsafe/deref diagnostics
layout and ABI fingerprints
```

qtlc1 must not implement allocator-backed pointer libraries, smart pointer
runtime behavior, or OS memory management.

### Tuple, Array, And Aggregate Shapes

```text
(A, B)
[T; N]
[]T
Type { ... }
{ field: value }
```

qtlc0 fixtures already exercise tuple types, fixed arrays, dynamic array/view
spellings, and typed aggregate literals. qtlc1 must model these as compiler
type/layout facts:

```text
tuple element order and layout
fixed array length and element layout
dynamic view/slice carrier identity
typed data shape field layout
anonymous record field identity
```

The empty tuple spelling is not qtlc1 style. Treat it only as legacy bootstrap
compatibility for `void`.

qtlc1 must not implement collection storage or reflection for these shapes.

### Collections As Compiler Type Facts

```text
Vec<T>
Vector<T>
VecBuilder<T>
VectorBuilder<T>
SmallVector<T>
SliceBuf<T>
Map<K, V>
HashMap<K, V>
BTreeMap<K, V>
OrderedMap<K, V>
IndexMap<K, V>
Set<T>
HashSet<T>
BTreeSet<T>
OrderedSet<T>
Queue<T>
Iterator<T>
```

qtlc0 has broad collection recognition in ownership/backend facts. qtlc1 only
needs the subset its compiler source uses:

```text
Vec<T>
Map<K, V>
Set<T>
Slice<T>
Array<T, N> or [T; N]
```

Full collection APIs, optimized storage, iterators, guards, cursors, queues,
and pool-backed containers are qtlc2/qtlc3 work.

### Quantum Core Types

```text
qubit
qreg
qreg<N>
qresult
bit
bits
bits<N>
qfunc
qfunc(...)
qany
```

qtlc0 TIR currently maps `qubit`, `qresult`, `qreg`, `bit`, `bits`, and
`qfunc` to QIR-style ABI classes. qtlc1 docs also reserve `qany` for the final
dynamic quantum value model.

qtlc1 must implement compiler facts:

```text
quantum type identity
ownership/no-clone facts
measurement result facts
reset/release policy facts
gate/capability facts
target/provider support facts
QMIR/QTIR product identity
```

qtlc1 must not implement a simulator, cloud provider SDK, quantum runtime
execution, or hardware job manager.

## Literal Rules

qtlc1 follows the literal rules locked in `QTLC1_FULL_IMPLEMENTATION_SCOPE.md`.

Required core literals:

```text
integer literals
float literals when compiler source needs them
bool literals
string literals
bytes literals if compiler source needs them
void/no-value return type
none
nullptr
nullref
fn(...) => ...
qubit |0>
qreg |0000>
bit 1
bits 0b10101010
qfunc { ... }
Type { ... }
This { ... }
this { ... }
[1, 2, 3]
[]
{ field: value }
{ "key": value }
```

Meaning:

```text
[1, 2, 3]
  Vec<T> by default when no fixed-array target exists

[]
  requires target type or clear context

Array<T, N> or [T; N] target
  fixed array

{ field: value }
  anonymous record

{ "key": value }
  dynamic Map

Type { field: value }
  typed object/data literal
```

### Literal Target Compatibility Matrix

Literal typing must use the central type catalog for target validation.
`LiteralTypingPolicy` only owns default choices such as untyped integer -> i32
and untyped float -> f64. It must not list every supported type.

Normal scalar literals must not silently become quantum, pointer, reference, or
function values. Those families have explicit literal spellings:

```qn
let q: qubit = qubit |0>
let r: qreg<4> = qreg |0000>
let b: bit = bit 1
let bs: bits<8> = bits 0b10101010
let f: qfunc = qfunc { ... }

let p: ptr<i32> = nullptr
let r: ref<User> = nullref

let fnc: fn(i64) -> i64 = fn(x) => x + 1
```

Compatibility rules:

```text
integer literal
  may target: integer primitives, int/uint aliases, BigInt, BigUint,
              dec/dec64/dec128/dec256, fixed, and Option<same>
  must reject: string/bytes/quantum/function/pointer/reference targets

float literal
  may target: f16/f32/f64/f128, dec/dec64/dec128/dec256, fixed,
              and Option<same>
  must reject: integer-only, text/bytes, quantum, function, pointer/reference
               targets unless an explicit conversion exists later

bool literal
  may target: bool and Option<bool>

string literal
  may target: string, str, Option<string>, Option<str>

bytes literal
  may target: bytes, BytesView, BytesMutView, and Option<same>

byte literal
  may target: byte, u8, Option<byte>, Option<u8>

none
  may target: Option<T> only, with a valid payload shape

Vec literal
  may target: Vec or Vec<T>; [] still requires target type or clear context

fixed array literal
  may target: Array, Array<T,N>, or [T; N]

map literal
  may target: Map or Map<K,V>; quoted-key maps may infer Map<string,T>

record/object literal
  may target: custom data/object type or anonymous record
  must reject: known builtin scalar/quantum/function/pointer/reference targets

nullptr
  may target: nullable/raw pointer families only

nullref
  may target: reference families only where the type explicitly permits null

function literal
  may target: fn(...) and fn(...) -> T shapes only

quantum literals
  qubit |0> targets qubit
  qreg |0000> targets qreg<N>
  bit 1 targets bit
  bits 0b... targets bits<N>
  qfunc { ... } targets qfunc
```

qtlc1 may add the literal-kind names and target compatibility skeleton before
lowering/runtime support exists. Full quantum execution, pointer runtime
behavior, and function closure lowering are later compiler/runtime work.

Package/domain literals such as `json { ... }`, `bson { ... }`, SQL, HTTP
route literals, and binary codec literals are not qtlc1 scope.

## Operator Scope

qtlc1 must implement syntax and typing for core operators used by compiler
source.

### Arithmetic

```text
+  -  *  /  %
```

Applies to integer and supported numeric facts. Decimal/fixed/wide/complex
operator support in qtlc1 can be limited to type checking and diagnostics unless
compiler source needs constant folding.

Planned later:

```text
**
```

`**` is power/exponentiation. It should be numeric-only and should not be
treated as syntax sugar for repeated multiplication unless the type checker has
the right overflow/precision policy for the target type.

### Comparison

```text
==  !=  <  <=  >  >=
```

qtlc1 must distinguish equality-capable types from ordered types. Complex
numbers are not ordered.

### Logical

```text
&&  ||  !
```

Operands must be `bool`.

### Bitwise And Shifts

```text
&  |  ^  ~  <<  >>
```

Applies to integer/bit facts. Quantum `bit` and `bits<N>` need explicit
quantum-lowering facts before being treated like normal integer bits.

### Assignment

```text
=  +=  -=  *=  /=  %=  &=  |=  ^=  <<=  >>=
```

qtlc1 should support these only where mutable locals/fields are legal.

### Access, Call, And Index

```text
.      field/member access
()     call
[]     index
?      Option/Result propagation
as     explicit cast/conversion
```

qtlc1 must make index typing stable for `Vec<T>`, `[T; N]`, `Slice<T>`,
`string`, and `bytes` facts used by compiler source.

Current range syntax:

```text
..     half-open range
..=    inclusive range
```

`??` remains deferred and is not currently accepted by qtlc0. Ranges produce
typed range facts for loops and slicing.

`?:` is intentionally not part of the preferred Quantum style. Use expression
`if` instead:

```qn
let label = if ok { "ready" } else { "failed" }
```

`++` and `--` are supported prefix/postfix mutation operators. `===` and `%%`
remain intentionally unsupported; use typed equality `==`, explicit identity
helpers when needed, and named math APIs for special modulo semantics.

### Syntax Body Operator

```text
=>
```

`=>` is kept as a syntax-level operator for expression bodies, lambda bodies,
and view-property bodies. It is not a normal binary expression operator and
should not be typed through `binaryOperator(...)`.

### Address And Dereference

```text
&value
&mut value
*ptr
```

qtlc1 must parse and type-check these enough for compiler source and low-level
facts. Raw dereference should require an unsafe/capability gate once that gate
exists in qtlc1.

## Memory Model Facts

qtlc1 needs compiler facts, not a runtime memory implementation.

Required:

```text
target pointer width
target pointer alignment
size and alignment per builtin type
fixed array size and element layout
struct/data shape field offsets
ABI return kind: register, void, sret
pointer/reference nullability
copy/move/drop policy facts
borrow/mutability facts
string/bytes runtime-managed carrier fact
Option/Result carrier fact
quantum ownership/no-clone facts
```

Not qtlc1:

```text
allocator implementation
arena implementation
garbage collector
reference counting runtime
thread-safe smart pointer runtime
OS virtual memory
embedded HAL memory devices
stdlib memory package
```

## Canonical qtlc1 Source Style

New qtlc1 source should prefer:

```text
bool, not Bool
string or str, not Text/String
bytes, BytesView, or BytesMutView, not Bytes
void, not Unit or empty-tuple spelling
byte for byte-data values; u8 for arithmetic unsigned 8-bit integers
i32/i64/u64/usize are canonical; int/uint are the only compatibility aliases
Vec<T>, Map<K, V>, Set<T>, Slice<T> only when compiler source needs them
T? for optional values
plain T value lifting into T? and none
```

Only these compatibility aliases remain in qtlc1:

```text
int -> i64
uint -> u64
```

All other old qtlc0/bootstrap spellings are rejected by the qtlc1 core catalog.

## Implementation Commands

```text
POST-QTLC1-CORE-001
  add builtin primitive catalog and compatibility alias table

POST-QTLC1-CORE-002
  add numeric, string, bytes, bool, void, option, and record literal typing

POST-QTLC1-CORE-003
  add operator precedence and core operator typing rules

POST-QTLC1-CORE-004
  add pointer, reference, array, slice, and nullability facts

POST-QTLC1-CORE-005
  add target data-layout and ABI return facts

POST-QTLC1-CORE-006
  add Option/Result carrier facts and `?` propagation typing

POST-QTLC1-CORE-007
  add Vec/Map/Set/record literal target-typing facts

POST-QTLC1-CORE-008
  add quantum core type metadata facts for qubit/qreg/qresult/qfunc/qany

POST-QTLC1-CORE-009
  add explicit quantum/function/pointer/reference literal target skeleton:
  qubit/qreg/bit/bits/qfunc, nullptr, nullref, and fn(...) => ...
```

Each command must leave `qtlang/compiler` qtlc0-checkable and must not add
runtime/stdlib implementation code.
