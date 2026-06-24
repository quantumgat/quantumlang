# Quantum OOP Surface And Boundary Design

Status: language design contract for qtlc0 proofs and qtlc1/qtlc2/qtlc3 implementation

This document locks the intended QuantumLang OOP surface model. The goal is to
support simple user code and very complex API/library/system design without
turning every type into one large class.

The core rule is surface separation:

```text
Type/class/resource/actor/service
  what the thing is

impl
  normal behavior

impl view
  read-only property API

role
  behavior contract

extend Type as Role
  role implementation

extend<T: Role<T>> T
  bounded helper methods for every type satisfying a role

context
  capability-aware behavior

extract
  typed input binding

unsafe/extern
  low-level boundary
```

## Surface List

```qn
pub Type { ... }              # value/data shape
pub class Type { ... }        # identity object with private mutable state
pub resource Type { ... }     # owned external resource with drop/close safety
pub actor Type { ... }        # isolated concurrent object
pub service Type { ... }      # lifecycle-managed application component

impl Type { ... }             # normal constructors and methods
impl view Type { ... }        # read-only computed properties
role Name { ... }             # trait/interface-style contract
extend Type as Role { ... }   # implement a role for one type
extend<T: Role<T>> T { ... }  # add bounded helper methods
context Type { ... }          # capability/context behavior
extract Type { ... }          # typed extraction/binding behavior
unsafe Type { ... }           # raw low-level methods
extern { ... }                # OS/native ABI boundary
```

## Value Type

Use `pub Type { ... }` for data, compiler records, DTOs, BSON/JSON documents,
AST nodes, tokens, spans, configs, and other value objects.

```qn
pub User {
  id: UserId
  name: string
  email: string?
  active: bool = true
}
```

Capabilities:

```text
fields
field default values
auto constructor
copy/update with this { ... }
direct field access inside impl/view/context/extract/extend if boundary allows it
```

Construction:

```qn
let user1 = User(id, name)
let user2 = User { id, name }
let user3 = User(id, name, active: false)
```

Auto-constructor contract:

```qn
pub User {
  id: i64
  name: string
  role: i64 = 10
  isActive: bool = false
  isEmailVerified: bool = false
}

let a = User(1, "Ali")
let b = User(1, "Ali", role: 20)
let c = User(1, "Ali", isActive: true)
let d = User(1, "Ali", 20, true, false)

let e = User { 1, "Ali" }
let f = User { 1, "Ali", role: 20 }
let g: User = { 1, "Ali" }
let h: User = { 1, "Ali", role: 10, isEmailVerified: true }
let i: User = {
  id: 1
  name: "Ali"
  role: 10
}
let j: User = { id: 1, name: "Ali", role: 10 }
```

Rules:

```text
pub Type { ... } creates the structural auto-constructor even without impl Type.
Fields without defaults are required.
Fields with defaults are optional constructor arguments.
Positional arguments bind fields in declaration order.
Named arguments may skip optional fields.
Positional arguments must come before named arguments.
Multiline object literals may use newline separators; commas are optional.
Inline object literals must use commas between fields.
If all fields have defaults, Type() is valid.
If required fields exist, Type() is invalid unless a real custom constructor
matches it.
Auto-construction lowers to direct native aggregate stores:
no reflection, no named-argument map, no runtime constructor table, no dynamic
dispatch.
```

Custom constructors are still valid when a type needs validation,
normalization, or computed fields:

```qn
impl User {
  pub User(id: i64, name: string) =
    This {
      id
      name
      role: 100
      isActive: true
    }
}
```

Inside an implementation:

```text
This { ... } creates a fresh aggregate and fills omitted default fields.
this { ... } copies and updates the current value.
This(...) calls a constructor.
This() inside the same empty constructor is recursion and must be rejected.
```

Boundary:

```text
public fields are visible outside the module/package according to export rules
private fields are visible only inside the owning module/package boundary
foreign package extensions cannot read private fields
```

## Class

Use `pub class Type { ... }` for identity objects, private mutable state,
lifecycle, dynamic role values, and object-style APIs.

```qn
pub class Wallet {
  private balance: decimal = 0.dec
  owner: AccountId

  Wallet(owner: AccountId) =
    This { owner }

  amount => balance

  deposit(value: decimal) mut {
    balance += value
  }

  withdraw(value: decimal) mut -> Result<void, WalletError> {
    if balance < value {
      return Err(WalletError.insufficientBalance)
    }

    balance -= value
  }
}
```

Capabilities:

```text
identity
private state
mut methods
lifecycle hooks later
role implementation
dynamic dispatch through any Role later
```

Boundary:

```text
private state is not visible to external extensions
mutation requires mut
inheritance is allowed only through explicit open/is rules
composition with has is preferred over inheritance
```

## Resource

Use `resource` for files, sockets, memory handles, GPU handles, locks, database
connections, and any value that must be closed or dropped safely.

```qn
pub resource File {
  fd: RawFd

  File(path: string) -> Result<This, IoError> {
    let fd = os.open(path)?
    This { fd }
  }

  read() -> Result<bytes, IoError> =
    os.read(fd)

  close!() move {
    os.close(fd)
  }

  drop {
    if fd >= 0 {
      os.close(fd)
    }
  }
}
```

Capabilities:

```text
move-only operations
drop hook
close!/consume methods
using/defer integration later
```

Boundary:

```text
unsafe raw handle access must live in unsafe Type or unsafe extend
resource ownership cannot be copied unless explicitly clone-safe
```

## Actor

Use `actor` for isolated concurrent state.

```qn
pub actor PriceFeed {
  prices: Map<Symbol, decimal>

  update(symbol: Symbol, price: decimal) {
    prices[symbol] = price
  }

  get(symbol: Symbol) -> decimal? {
    prices.get(symbol)
  }
}
```

Capabilities:

```text
isolated mutable state
async message calls
no shared data race by default
```

Boundary:

```text
direct state access is actor-local
external calls cross actor boundary and may require await
```

## Service

Use `service` for lifecycle-managed application components.

```qn
pub service HttpServer {
  router: Router
  port: u16

  HttpServer(port: u16) =
    This { port, router: Router() }

  route(path: string, handler: Handler) mut {
    router.add(path, handler)
  }

  start() async {
    net.listen(port, router).await
  }

  stop() async {
    net.shutdown().await
  }
}
```

Capabilities:

```text
start/stop lifecycle
dependency injection later
supervised runtime integration later
```

Boundary:

```text
service dependencies must be declared
compiler/runtime should reject missing dependencies before runtime
```

## Impl

`impl Type { ... }` is for normal constructors, methods, mutation, copy/update,
and helpers.

```qn
impl User {
  User(id: UserId, name: string) =
    This { id, name }

  rename(value: string) mut {
    name = value
  }

  withName(value: string) =
    this { name: value }
}
```

Rules:

```text
This { ... } creates a fresh value
this { ... } copies/updates the current value
This(...) calls a constructor
this.field and bare field both work inside methods
mutation requires mut
final expression return is allowed
return is still allowed for early exits
```

Boundary:

```text
impl in the owning module/package may access private fields
foreign impl/extend cannot access private fields
public methods must be exported/imported across module boundaries
```

## Impl View

`impl view Type { ... }` is for read-only computed properties.

```qn
impl view HttpCtx<...Caps> {
  method => request.method
  path => request.uri.path
  headers => request.headers
  body => request.body
}
```

Usage:

```qn
let path = ctx.path
let method = ctx.method
```

Capabilities:

```text
read fields
call pure/read-only methods
computed properties
zero-cost inline lowering where possible
```

Forbidden:

```text
mutation
hidden I/O
hidden allocation unless explicitly allowed by effect/capability rules later
unsafe access
```

Boundary:

```text
public view properties are exported API
private/package view properties remain inside boundary
conflicting view names from multiple roles/extensions require explicit role view
```

## Role

`role` is the core contract system. It covers trait/interface-style design.

```qn
pub role Repository<T> {
  find(id: Id) -> T?
  save(value: T) mut -> Result<void, DbError>
}
```

Object-safe role alias can be allowed later:

```qn
pub interface Repository<T> {
  find(id: Id) -> T?
  save(value: T) mut -> Result<void, DbError>
}
```

Meaning:

```text
role
  general contract, static or dynamic

interface
  optional alias for object-safe role

some Role
  static dispatch, fastest

any Role
  dynamic dispatch, runtime role object
```

Boundary:

```text
roles must be exported/imported like other symbols
role conformance must be checked at compile time
role methods do not get private field access by themselves
```

## Extend As Role

`extend Type as Role { ... }` implements a role for a specific type.

```qn
pub role Verifiable {
  verify() -> bool
}

pub Transaction {
  from: Address
  signature: Signature
}

pub extend Transaction as Verifiable {
  verify() -> bool {
    signature.verify(from.publicKey)
  }
}
```

Capabilities:

```text
implements role contract
can be public/package/private
can be static-dispatched through some Role
can be dynamic-dispatched through any Role later
```

Boundary:

```text
same module/package extension may access allowed fields
foreign package extension cannot access private fields
pub extend becomes public API only when exported
package extend is visible only inside package
private extend is local
unsafe extend requires unsafe boundary
```

Multiple roles are allowed:

```qn
pub extend HttpCtx<...Caps> as RequestContext { ... }
pub extend HttpCtx<...Caps> as CapabilityContext { ... }
pub extend HttpCtx<...Caps> as ExtractContext { ... }
```

Role composition is allowed later:

```qn
pub role HttpContext =
  RequestContext + CapabilityContext + ExtractContext

pub extend HttpCtx<...Caps> as HttpContext
```

## Blanket Extension

`extend<T: Role<T>> T { ... }` adds helper methods to every type satisfying a
bound. This is needed for clean APIs like BSON, JSON, binary codecs, math, and
formatting.

Good:

```qn
pub extend<T: BsonCodec<T>> T {
  toBson() -> Result<BsonBytes, BsonError>
  toBson(mode: BsonCodecMode) -> Result<BsonBytes, BsonError>
  toBsonExact() -> Result<BsonBytes, BsonError>
}
```

Bad:

```qn
pub extend<T> T {
  toBson() -> Result<BsonBytes, BsonError>
}
```

The unbounded form adds methods to every type and can create conflicts.

Boundary:

```text
public blanket extensions must be bounded
package can define blanket helpers for roles it owns
extension is visible only when imported/exported
inherent impl methods win over extension methods
two matching extension methods with same name are ambiguous unless one is more specific
compiler must report ambiguity instead of guessing
```

Recommended coherence rule:

```text
A package may publish an extension when it owns at least one of:
  the role
  the extended type
  the capability namespace

Foreign type + foreign role blanket extension should be rejected or package-local only.
```

## BSON Codec Example

Define the contract:

```qn
pub role BsonCodec<T> {
  encode(value: T, mode: BsonCodecMode) -> Result<BsonBytes, BsonError>
  decode(bytes: BinaryView, mode: BsonCodecMode) -> Result<T, BsonError>
}
```

Add user-facing helpers:

```qn
pub extend<T: BsonCodec<T>> T {
  toBson() -> Result<BsonBytes, BsonError> =
    BsonCodec<T>.encode(this, BsonCodecMode.default)

  toBson(mode: BsonCodecMode) -> Result<BsonBytes, BsonError> =
    BsonCodec<T>.encode(this, mode)

  toBsonExact() -> Result<BsonBytes, BsonError> =
    BsonCodec<T>.encode(this, BsonCodecMode.exact)
}
```

Use owned bytes for newly encoded data:

```text
BsonBytes or bytes
  correct return type for newly encoded BSON

BinaryView
  correct for borrowed input or viewing existing bytes
```

User code:

```qn
let bytes = user.toBson()?
let strict = user.toBsonExact()?
let decoded = BsonCodec<User>.decode(bytes.view, BsonCodecMode.default)?
```

Implement for a type:

```qn
pub User {
  id: UserId
  name: string
}

pub extend User as BsonCodec<User> {
  encode(value: User, mode: BsonCodecMode) -> Result<BsonBytes, BsonError> {
    bson<User> {
      id: value.id
      name: value.name
    }.bytes(mode)
  }

  decode(bytes: BinaryView, mode: BsonCodecMode) -> Result<User, BsonError> {
    bson.decode<User>(bytes, mode)
  }
}
```

## Complex Number Example

Unbounded complex extension is wrong:

```qn
pub extend<C, R> C {
  real() -> R
  imag() -> R
}
```

It would add complex methods to every type. Use a bound:

```qn
pub role RealScalar {
  sqrt() -> This
  atan2(x: This) -> This
}

pub role ComplexLike<C, R> {
  real() -> R
  imag() -> R
  fromParts(real: R, imag: R) -> C
}

pub extend<C: ComplexLike<C, R>, R: RealScalar> C {
  real() -> R
  imag() -> R

  conj() -> C =
    C.fromParts(real(), -imag())

  norm() -> R =
    real() * real() + imag() * imag()

  abs() -> R =
    norm().sqrt()

  arg() -> R =
    imag().atan2(real())

  arg(policy: ComplexBranchPolicy) -> Result<R, ComplexError>

  sqrt() -> Result<C, ComplexError>
  exp() -> C
  ln() -> Result<C, ComplexError>
  pow(exponent: C) -> Result<C, ComplexError>
  div(right: C) -> Result<C, ComplexError>

  format() -> string
  format(format: ComplexFormat) -> string
}
```

Implement concrete complex types:

```qn
pub complex128 {
  re: f64
  im: f64
}

pub extend complex128 as ComplexLike<complex128, f64> {
  real() -> f64 = re
  imag() -> f64 = im

  fromParts(real: f64, imag: f64) -> complex128 =
    complex128 { re: real, im: imag }
}
```

User code:

```qn
let z = complex128(3.0, 4.0)
let r = z.abs()
let c = z.conj()
let text = z.format()
```

## Context

`context Type { ... }` is for capability-aware behavior. It is not just normal
`impl`; it can require or provide capabilities.

```qn
pub HttpCtx<...Caps> {
  request: HttpRequest
  slots: CapSlots<...Caps>
}

context HttpCtx<...Caps> {
  has<T> => Caps.has<T>()

  require<T>() -> Result<T, HttpError> {
    slots.require<T>(HttpError.badRequest("missing capability"))
  }

  with<T>(value: T) =
    this { slots[T]: value }
}
```

Capabilities:

```text
read declared capabilities
require capabilities
add scoped context data
flow capability facts through compiler/runtime
```

Boundary:

```text
context APIs must declare required capabilities
cross-module context use requires import/export
capability slots are not globally visible
unsafe capability escape must use unsafe context/extend
```

## Extract

`extract Type { ... }` converts external input into typed values.

```qn
extract HttpCtx<...Caps> {
  path<T>(name: string) -> Result<T, HttpError> =
    slots.path<T>(request, name)

  query<T>(name: string) -> Result<T, HttpError> =
    slots.query<T>(request, name)

  body<T>() -> Result<T, HttpError> =
    slots.body<T>(request)
}
```

Capabilities:

```text
HTTP path/query/header/body extraction
JSON/BSON/binary decode
CLI argument binding
database row mapping
compiler token/AST binding
```

Boundary:

```text
extractors must return typed success/failure
extractors must not silently throw hidden runtime errors
public extractors are framework API and must be exported
private extractors stay local to the package/module
```

## Unsafe And Extern

Unsafe surfaces isolate raw power.

```qn
unsafe extend HttpCtx<...Caps> as RawHttpCtx {
  slotPtr<T>() -> Ptr<T> {
    slots.rawPtr<T>()
  }
}
```

```qn
extern "c" {
  open(path: *const u8, flags: i32) -> RawFd
}
```

Boundary:

```text
unsafe methods require unsafe call site or unsafe enclosing block
unsafe extension is never imported through normal prelude
extern functions must declare ABI, ownership, and error model
```

## Conflict Rules

If one type has two role extensions with the same property name:

```qn
role TextId {
  id -> string
}

role NumericId {
  id -> u64
}

extend User as TextId {
  id => name
}

extend User as NumericId {
  id => numericId
}
```

Allowed:

```qn
let text: TextId = user
let number: NumericId = user

text.id
number.id
```

Ambiguous:

```qn
user.id
```

Required:

```qn
user.as<TextId>().id
user.as<NumericId>().id
```

Resolution order:

```text
1. fields and inherent impl/view members on the concrete type
2. explicitly selected role view
3. imported extension methods with a single best match
4. ambiguity error
```

The compiler must never guess between two equally valid extension methods.

## Visibility And Module Boundary

QuantumLang module boundary stays strict:

```text
inside one module
  files can see module-private implementation details

outside the module
  caller needs export/import through mod.qn and module.toml dependency

inside package
  package extend/context/extract can be shared according to package visibility

outside package
  only pub exported surfaces are visible
```

Access levels:

```qn
pub extend Type as Role { ... }      # public API
package extend Type as Role { ... }  # package-only API
private extend Type as Role { ... }  # local API
unsafe extend Type as Role { ... }   # unsafe API
```

Private field rule:

```text
owning module/package extension
  may access private fields if visibility allows it

foreign package extension
  cannot access private fields

role contract
  does not grant private access by itself
```

## Performance Contract

These surfaces are language/API organization, not runtime overhead by default.

```text
impl
  direct call or inline

impl view
  property syntax, getter lowering, inline when possible

extend<T: Role<T>> T
  static dispatch for bounded generic helpers

some Role
  static dispatch, optimized like generic code

any Role
  dynamic dispatch, explicit runtime cost

context/extract
  static where possible; runtime only when framework needs it

resource
  compiler-known ownership/drop; no hidden GC required

actor/service
  runtime-managed by design, explicit boundary
```

## Stage Plan

qtlc0:

```text
parse and build proof for impl view, role, extend, context, extract
basic direct method/property attachment
no full dynamic dispatch
no final capability runtime
no full extractor framework runtime
```

qtlc1:

```text
compiler-only use of these surfaces where needed for qtlc2
role/type metadata model
clear diagnostics and module boundaries
no full stdlib/runtime implementation
```

qtlc2:

```text
real role checking
bounded blanket extension checking
context/extract compiler semantics
qcap/qidx metadata for surfaces
```

qtlc3:

```text
full class/resource/actor/service model
dynamic any Role
macro/derive
framework/runtime integration
package/plugin/LSP/tooling support
```

## Final Rule

Use the smallest surface that matches the job:

```text
data only
  pub Type

identity and private mutable state
  pub class Type

normal behavior
  impl Type

read-only property API
  impl view Type

contract
  role Name

contract implementation
  extend Type as Role

role-bound helper methods
  extend<T: Role<T>> T

capability/context behavior
  context Type

input binding
  extract Type

raw power
  unsafe Type / unsafe extend / extern
```

This keeps QuantumLang short for end users, strong for framework authors, and
safe enough for system-level code.
