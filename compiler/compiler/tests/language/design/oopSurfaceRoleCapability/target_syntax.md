# Target Syntax

This file is the target source design for the future compiled proof. It uses
the intended QuantumLang syntax, including syntax not yet implemented in qtlc0.

## Role Families

All role conformances are surface-capable. The same role machinery is shared by
`impl`, `impl view`, `context`, `extract`, and `extend`.

```qn
pub role HttpLifecycle<T> {
  withTrace(trace: TraceId) -> T
  freeze() -> Frozen<T>
}

pub role HttpReadableView<T> {
  path() -> string
  method() -> HttpMethod
  traceId() -> TraceId
}

pub role HttpDebugView<T> {
  debugSummary() -> string
}

pub role CapabilityContext<...Caps> {
  has<T>() -> bool
  require<T>() -> Result<T, IntoHttpError>
}

pub role RequestContext<T> {
  requestId() -> RequestId
  clientIp() -> string
}

pub role BodyExtractor<...Caps> {
  query<T>(name: TextView = TextView("")) -> Result<T, IntoHttpError>
  body<T>() -> Result<T, IntoHttpError>
  valid<T>() -> Result<Valid<T>, IntoHttpError>
}

pub role SessionExtractor<...Caps> {
  session<T>() -> Result<Session<T>, IntoHttpError>
  sessionData<T>() -> Result<T, IntoHttpError>
}

pub role Authenticated<T> {
  user() -> Result<AuthUser, IntoHttpError>
}

pub role AuditRole<T> {
  auditEvent(action: TextView) -> AuditEvent
}
```

## Data And Pack Model

```qn
pub HttpCtx<...Caps> {
  request: HttpRequest
  slots: CapSlots<...Caps>
  trace: TraceId
}

pub CapSlots<...Caps> {
  storage: CapabilityStorage<...Caps>
}

pub Valid<T> {
  value: T
  source: string
}

pub Session<T> {
  value: T
  id: SessionId
}
```

Required pack behavior:

```qn
HttpCtx<AuthUser, Session<UserSession>, CsrfToken, JsonBody<CreateUserBody>>
HttpCtx<TraceCap, ...Caps>
CapSlots<...Caps>
CapabilityContext<...Caps>
```

The spread `...Caps` must preserve order, identity, type hashes, and native
layout determinism.

Every capability carried by a proof pack must explicitly satisfy
`HttpCapability`:

```qn
extend AuthUser : pub HttpCapability { ... }
extend CsrfToken : pub HttpCapability { ... }
extend<T> Session<T> : pub HttpCapability { ... }
extend<T> JsonBody<T> : pub HttpCapability { ... }
```

## Surface Role Stress Syntax

The proof must cover the exact high-power surface shape:

```qn
impl HttpCtx<...Caps> :
  Omg<HttpCtx<...Caps>>,
  pub PublicImplRole<HttpCtx<...Caps>>,
  internal InternalImplRole<...Caps>,
  private PrivateImplAudit
{
}

impl view HttpCtx<...Caps> :
  OmgView<HttpCtx<...Caps>>,
  pub PublicViewRole<HttpCtx<...Caps>>,
  internal DebugViewRole
{
}

context HttpCtx<...Caps> :
  OmgContext<...Caps>,
  pub TestContext<...Caps>,
  internal InternalContext<HttpCtx<...Caps>>
{
}

extract HttpCtx<...Caps> :
  OmgExtract<...Caps>,
  pub PublicExtract<...Caps>,
  internal InternalExtract<HttpCtx<...Caps>>
{
}

extend HttpCtx<...Caps> :
  OmgExtend<HttpCtx<...Caps>>,
  pub PublicExtend<HttpCtx<...Caps>>,
  private PrivateExtendAudit
{
}
```

The proof must also cover generic parameter plus pack spread:

```qn
impl<T, ...Caps> HttpCtx<T, ...Caps> :
  pub OmgImpl<HttpCtx<T, ...Caps>>,
  internal OmgImplPack<T, ...Caps>
where
  T: HttpCapability,
  ...Caps: HttpCapability
{
}

impl view<T, ...Caps> HttpCtx<T, ...Caps> :
  pub OmgView<HttpCtx<T, ...Caps>>,
  internal OmgViewPack<T, ...Caps>
where
  T: HttpCapability,
  ...Caps: HttpCapability
{
}

context<T, ...Caps> HttpCtx<T, ...Caps> :
  pub OmgContext<T, ...Caps>,
  internal TestContext<HttpCtx<T, ...Caps>>
where
  T: HttpCapability,
  ...Caps: HttpCapability
{
}

extract<T, ...Caps> HttpCtx<T, ...Caps> :
  pub OmgExtract<T, ...Caps>,
  internal TestExtract<HttpCtx<T, ...Caps>>
where
  T: HttpCapability,
  ...Caps: HttpCapability
{
}

extend<T, ...Caps> HttpCtx<T, ...Caps> :
  pub OmgExtend<HttpCtx<T, ...Caps>>,
  private OmgExtendAudit<T, ...Caps>
where
  T: HttpCapability,
  ...Caps: HttpCapability
{
}
```

Role-list normalization rule:

```text
`Role` means `private Role`.
`pub Role`, `internal Role`, and `private Role` are per-role flags.
Every surface uses the same role-list parser and the same conformance record.
```

## Impl Surface With Roles

```qn
impl HttpCtx<...Caps> :
  pub HttpLifecycle<HttpCtx<...Caps>>,
  internal RequestContext<HttpCtx<...Caps>>
{
  pub HttpCtx(request: HttpRequest, slots: CapSlots<...Caps>) =
    This {
      request
      slots
      trace: TraceId.new()
    }

  pub withTrace(trace: TraceId) -> HttpCtx<...Caps> {
    return HttpCtx<...Caps> {
      request: this.request
      slots: this.slots
      trace
    }
  }

  pub freeze() -> Frozen<HttpCtx<...Caps>> {
    return Frozen<HttpCtx<...Caps>>(this)
  }

  internal requestId() -> RequestId {
    return this.request.id
  }

  internal clientIp() -> string {
    return this.request.clientIp
  }
}
```

## Impl View Surface With Roles

```qn
impl view HttpCtx<...Caps> :
  pub HttpReadableView<HttpCtx<...Caps>>,
  internal HttpDebugView<HttpCtx<...Caps>>
{
  pub path: string => request.path
  pub method: HttpMethod => request.method
  pub traceId: TraceId => trace
  pub capCount: i64 => Caps.count
  pub empty: bool => Caps.count == 0

  internal debugSummary: string =>
    "{method}:{path}:caps={capCount}:trace={traceId}"
}
```

View role matching rule:

```text
role function `path() -> string` can be satisfied by view property `path: string`.
The lowered direct call is the same scalar/view getter, not dynamic dispatch.
```

## Context Surface With Roles

```qn
context HttpCtx<...Caps> :
  pub CapabilityContext<...Caps>,
  internal RequestContext<HttpCtx<...Caps>>
{
  pub has<T>() -> bool {
    return Caps.has<T>()
  }

  pub require<T>() -> Result<T, IntoHttpError> {
    return slots.require<T>()
  }

  internal requestId() -> RequestId {
    return request.id
  }

  internal clientIp() -> string {
    return request.clientIp
  }
}
```

Context receiver rule:

```text
Bare `request`, `slots`, and `trace` inside context resolve to `this.request`,
`this.slots`, and `this.trace`.
```

## Extract Surface With Roles

```qn
extract HttpCtx<...Caps> :
  pub BodyExtractor<...Caps>,
  pub SessionExtractor<...Caps>,
  internal RequestContext<HttpCtx<...Caps>>
{
  pub path<T>(name: TextView) -> Result<T, IntoHttpError> {
    return slots.path<T>(request, name)
  }

  pub query<T>(name: TextView = TextView("")) -> Result<T, IntoHttpError> {
    if name.length == 0 {
      return slots.query<T>(request)
    }

    return slots.query<T>(request, name)
  }

  pub body<T>() -> Result<T, IntoHttpError> {
    return slots.body<T>(request)
  }

  pub valid<T>() -> Result<Valid<T>, IntoHttpError> {
    return body<T>().map(v => Valid<T>(v, "default"))
  }

  pub session<T>() -> Result<Session<T>, IntoHttpError> {
    return slots.session<T>()
  }

  pub sessionData<T>() -> Result<T, IntoHttpError> {
    return session<T>().map(s => s.value)
  }

  internal requestId() -> RequestId {
    return request.id
  }

  internal clientIp() -> string {
    return request.clientIp
  }
}
```

Extract receiver rule:

```text
Bare `request`, `slots`, `body<T>()`, and `session<T>()` inside extract resolve
through the extract receiver. This must be the same receiver model as context.
```

## Extend Surface With Roles

```qn
extend HttpCtx<...Caps> :
  pub Authenticated<HttpCtx<...Caps>>,
  private AuditRole<HttpCtx<...Caps>>
{
  pub user() -> Result<AuthUser, IntoHttpError> {
    return require<AuthUser>()
  }

  private auditEvent(action: TextView) -> AuditEvent {
    return AuditEvent(trace, action, TextView(request.path))
  }
}
```

Extend rule:

```text
Extension members are statically resolved. A direct call `ctx.user()` lowers to
one native function target. Private role methods remain callable only inside the
owning surface boundary.
```

## Same Role On Multiple Surfaces

This is allowed when the contracts are surface-qualified:

```qn
pub role HealthSignal<T> {
  healthy() -> bool
}

impl HttpCtx<...Caps> : pub HealthSignal<HttpCtx<...Caps>> {
  pub healthy() -> bool {
    return request.valid
  }
}

context HttpCtx<...Caps> : pub HealthSignal<HttpCtx<...Caps>> {
  pub healthy() -> bool {
    return request.valid && slots.valid
  }
}
```

Unqualified lookup must reject ambiguity if both are visible and neither is
selected by the call surface:

```qn
ctx.healthy()           // rejected if ambiguous
ctx.impl.healthy()      // explicit surface-qualified call
ctx.context.healthy()   // explicit surface-qualified call
```

Surface-qualified call syntax is part of this proof. Duplicate visible member
names across surfaces must be rejected for unqualified lookup and accepted only
when the source selects the surface explicitly.

## Same-Surface Merge

Multiple blocks for the same target and same surface are allowed. They merge
into one conformance group before duplicate checking and native symbol naming:

```qn
impl HttpCtx<...Caps> :
  pub HttpLifecycle<HttpCtx<...Caps>>
{
  pub withTrace(trace: TraceId) -> HttpCtx<...Caps> { ... }
}

impl HttpCtx<...Caps> :
  pub MutableHttpBuilder<HttpCtx<...Caps>>
{
  pub withHeader(name: TextView, value: TextView) -> HttpCtx<...Caps> { ... }
}
```

This must not create two independent target identities. Duplicate incompatible
members are compile errors after default and named arguments are normalized.

## Complex End-User API

```qn
createUser(
  ctx: HttpCtx<AuthUser, Session<UserSession>, CsrfToken, JsonBody<CreateUserBody>>
) -> Result<UserDto, IntoHttpError> {
  let user = ctx.user()?
  let body = ctx.valid<CreateUserBody>()?
  let session = ctx.sessionData<UserSession>()?

  return Ok(UserDto(user.id, body.value.name, session.tenantId))
}
```

Complex composition with spread:

```qn
secureRoute<...Caps>(
  ctx: HttpCtx<AuthUser, Session<UserSession>, ...Caps>
) -> Result<RouteReady<HttpCtx<AuthUser, Session<UserSession>, ...Caps>>, IntoHttpError> {
  let user = ctx.user()?
  let session = ctx.sessionData<UserSession>()?

  if ctx.has<CsrfToken>() == false {
    return Err(IntoHttpError.forbidden(TextView("missing csrf")))
  }

  return Ok(RouteReady(ctx.withTrace(TraceId.new()), user, session))
}
```

## Advanced High-Level API Surface

The proof also needs the API forms that real users will compose in larger
frameworks.

```qn
pub role MutableHttpBuilder<T> {
  withHeader(name: TextView, value: TextView) -> T
  setTrace(trace: TraceId) -> T
  consumeBody() -> Result<T, IntoHttpError>
}

pub role AsyncBodyExtractor<...Caps> {
  bodyStream<T>() -> Result<BodyStream<T>, IntoHttpError>
  chunk<T>() -> Result<BodyChunk<T>, IntoHttpError>
  upload<T>() -> Result<Uploaded<T>, IntoHttpError>
}

pub role HttpMiddleware<T> {
  before(ctx: T) -> Result<T, IntoHttpError>
  after(ctx: T) -> Result<T, IntoHttpError>
  around(ctx: T, next: Next<T>) -> Result<T, IntoHttpError>
  onError(error: IntoHttpError) -> IntoHttpError
}

pub role HttpAssociatedSpec<T> {
  type Body
  type Error
  const requiresAuth: bool
}

pub role HttpPackQueries<...Caps> {
  packUnique() -> bool
  capIndex<T>() -> i64
  allHttpCapabilities() -> bool
}
```

Default, named-argument, streaming, builder, middleware, and role-object calls:

```qn
pub highLevelApi(
  ctx: HttpCtx<AuthUser, Session<UserSession>, CsrfToken, JsonBody<CreateUserBody>>
) -> Result<UserDto, IntoHttpError> {
  let next =
    ctx
      .setTrace(TraceId { value: 99 })
      .withHeader(TextView("x-trace"), TextView("99"))
      .consumeBody()?

  let queryDefault = next.query<CreateUserBody>()?
  let queryNamed = next.query<CreateUserBody>(name: TextView("payload"))?
  let bodyLimited = next.body<CreateUserBody>(limit: 65536)?
  let stream = next.bodyStream<CreateUserBody>()?
  let ready = next.extend.around(next, Next<HttpCtx<AuthUser, Session<UserSession>, CsrfToken, JsonBody<CreateUserBody>>> {
    value: next
  })?

  let auth: Authenticated<HttpCtx<AuthUser, Session<UserSession>, CsrfToken, JsonBody<CreateUserBody>>> =
    ready

  if !ready.extend.packUnique() || ready.extend.capIndex<AuthUser>() < 0 {
    return Err(IntoHttpError.badRequest(TextView("capability pack invalid")))
  }

  if !ready.extend.requiresAuth || ready.extend.bodySpec().limitBytes < 1 {
    return Err(IntoHttpError.badRequest(TextView("route spec invalid")))
  }

  return Ok(UserDto(
    auth.user()?.id,
    bodyLimited.name,
    queryDefault.name.length + queryNamed.name.length + stream.backpressureWindow
  ))
}
```

Module-boundary syntax must be explicit in proof fixtures:

```qn
// package http.core
pub PublicRouteCarrier { value: i64 = 0 }

impl PublicRouteCarrier :
  pub PublicRouteRole<PublicRouteCarrier>,
  internal InternalRouteRole<PublicRouteCarrier>,
  private PrivateRouteRole<PublicRouteCarrier>
{
}

// package http.feature
// import http.core::{PublicRouteCarrier, PublicRouteRole}

pub externalUse(route: PublicRouteCarrier) -> i64 {
  return route.impl.publicRouteValue()
}
```

## Import, Export, Re-Export

Cross-module capability must use canonical symbol identity, not text names.

```qn
module http.capability

mod {
  roles
  model
  surfaces
}

export {
  roles.{HttpCapability, Authenticated, BodyExtractor, CapabilityContext}
  model.{AuthUser, UserSession, Session, CsrfToken, JsonBody, CreateUserBody}
  surfaces.{HttpAssociatedSpec, HttpPackQueries, MutableHttpBuilder, HttpMiddleware}
}
```

Facade re-export:

```qn
module http

import http.capability as capability
import http.core as core

export {
  capability.*
  core.{HttpCtx, CapSlots, CapabilityStorage}
}
```

End-user import with aliases:

```qn
module app.routes.users

import http.{HttpCtx, AuthUser, UserSession, Session as HttpSession, CsrfToken, JsonBody}
import http.{Authenticated, HttpCapability, HttpAssociatedSpec, HttpPackQueries}
import db.session.{Session as DbSession}

pub route(
  ctx: HttpCtx<AuthUser, HttpSession<UserSession>, CsrfToken, JsonBody<CreateUserBody>>
) -> Result<UserDto, IntoHttpError> {
  let user = ctx.extend.user()?

  if !ctx.extend.allHttpCapabilities() {
    return Err(IntoHttpError.badRequest(TextView("bad cap import")))
  }

  return Ok(UserDto(user.id, user.name, 0))
}
```

Required identity rule:

```text
HttpSession<UserSession> and DbSession<UserSession> are separate symbols even
when their local unqualified names would both be `Session`.
```
