# rpc-router Core Code Spec

## Intent

This spec defines the core library surface and internal code structure for `rpc-router`, a Rust library that routes JSON-RPC method calls to async Rust handler functions with typed resource extraction and optional typed parameter deserialization.

The core functionality covers:

- registration of handler functions under JSON-RPC method names
- immutable shared router state for concurrent async use
- typed resource storage and lookup for handler dependencies
- optional JSON-RPC params deserialization into strongly typed Rust inputs
- normalized execution of a JSON-RPC request into either a success payload or a structured library error
- preservation of request context such as `id` and `method` in call results

The visible public interface includes types and traits such as:

```rust
pub struct Router;
pub struct Resources;
pub struct ResourcesBuilder;

pub trait Handler<T, P, R>: Clone { ... }
pub trait IntoParams: DeserializeOwned + Send { ... }
pub trait IntoDefaultRpcParams: DeserializeOwned + Send + Default {}
pub trait FromResources: Clone + Send + Sync + 'static { ... }

pub struct RpcRequest { ... }
pub enum RpcId { ... }

pub type Result<T> = core::result::Result<T, Error>;
pub enum Error { ... }
```

Expected usage is centered around:

- building a `Router` from handler functions and optional base resources
- parsing or constructing an `RpcRequest`
- executing the request through `Router::call(...)` or `Router::call_with_resources(...)`
- receiving a `CallResult` that preserves JSON-RPC call context and contains either a serialized success value or a library error

This spec focuses on the library core only. It does not define transport integration details such as HTTP, WebSocket, CLI, or GUI bindings.

## Code Design

### Public module surface

The crate root exposes a flattened public API from submodules under `src/lib.rs`.

Core module groups are:

- `error`
  - library-wide error type and `Result<T>`
- `handler`
  - handler abstraction, wrapper types, and handler error adaptation
- `params`
  - parameter conversion from `Option<serde_json::Value>`
- `resource`
  - typed resource storage, extraction, and builders
- `router`
  - route registration and dispatch execution
- `rpc_id`
  - JSON-RPC id representation
- `rpc_message`
  - request and related message types
- `rpc_response`
  - response-side call result types
- `support`
  - internal support utilities not intended as the main public surface

The crate also re-exports derive macros:

```rust
pub use rpc_router_macros::RpcHandlerError;
pub use rpc_router_macros::RpcParams;
pub use rpc_router_macros::RpcResource;
```

These macros are convenience helpers only. The runtime design remains trait-driven.

### Router subsystem

`Router` is the immutable dispatch entry point for the library.

Primary responsibilities:

- hold the route table through an internal `RouterInner`
- hold shared base resources for all calls
- expose ergonomic call methods for request-based and route-based execution
- support per-call resource overlays without mutating router-wide state

Public execution paths:

```rust
impl Router {
    pub fn builder() -> RouterBuilder;
    pub async fn call(&self, rpc_request: RpcRequest) -> CallResult;
    pub async fn call_with_resources(&self, rpc_request: RpcRequest, additional_resources: Resources) -> CallResult;
    pub async fn call_route(&self, id: Option<RpcId>, method: impl Into<String>, params: Option<Value>) -> CallResult;
    pub async fn call_route_with_resources(
        &self,
        id: Option<RpcId>,
        method: impl Into<String>,
        params: Option<Value>,
        additional_resources: Resources,
    ) -> CallResult;
}
```

Execution flow:

- `Router` receives a request or raw route components.
- The router computes effective resources for the call:
  - if the router has no base resources, per-call resources are used directly
  - otherwise, per-call resources overlay base resources
- The call is delegated to `RouterInner`
- `RouterInner` resolves the method name to a registered handler wrapper
- The wrapper invokes the concrete handler and normalizes the output into the crate response shape

The route table stores dynamically dispatched handler wrappers rather than monomorphized handler functions at call time. This keeps the runtime dispatch uniform once routes are registered.

### Handler abstraction

The `Handler<T, P, R>` trait defines the normalized callable contract implemented by supported async handler functions.

Core properties:

- handlers are async and return a `Future<Output = Result<Value>>`
- handlers are cloneable to support route storage and invocation ergonomics
- handler inputs are normalized into:
  - zero or more resource-derived arguments
  - zero or one parameter argument derived from JSON-RPC `params`
- handlers can be converted into boxed dynamic wrappers through `into_dyn`

Representative shape:

```rust
pub trait Handler<T, P, R>: Clone
where
    T: Send + Sync + 'static,
    P: Send + Sync + 'static,
    R: Send + Sync + 'static,
{
    type Future: Future<Output = Result<Value>> + Send + 'static;

    fn call(self, rpc_resources: Resources, params: Option<Value>) -> Self::Future;

    fn into_dyn(self) -> Box<dyn RpcHandlerWrapperTrait>
    where
        Self: Sized + Send + Sync + 'static;
}
```

Supporting pieces in the handler subsystem are responsible for:

- extracting typed resources from `Resources`
- converting optional JSON values into the expected params type
- invoking the user async function
- serializing a successful return value into `serde_json::Value`
- converting application handler errors into `HandlerError`
- exposing dynamic wrapper behavior via `RpcHandlerWrapperTrait`

This design allows application functions with distinct signatures to participate in one runtime router.

### Resource subsystem

`Resources` is an immutable typed resource map used to satisfy handler dependencies.

Main types:

- `ResourcesBuilder`
- `Resources`

`ResourcesBuilder` responsibilities:

- hold a mutable staging map
- append typed resource values
- support typed lookup before build
- build an immutable `Resources` object

`Resources` responsibilities:

- store resources behind shared `Arc`-backed internals
- provide typed lookup by concrete type
- support overlay composition for per-call overrides
- remain cheap to clone

Effective model:

```rust
pub struct Resources {
    base_inner: Arc<ResourcesInner>,
    overlay_inner: Arc<ResourcesInner>,
}
```

Lookup order is:

- first `overlay_inner`
- then `base_inner`

This supports the common case where a router has global resources and a specific call wants to override one or more resource values without rebuilding the router.

Resource extraction for handlers is performed through `FromResources`. A resource type is expected to be cloneable and retrievable from the type map.

### Params subsystem

`IntoParams` defines how optional JSON-RPC params are converted into a concrete Rust input type.

Baseline behavior:

- `Some(value)` is deserialized with `serde_json::from_value`
- `None` returns `Error::ParamsMissingButRequested`

```rust
pub trait IntoParams: DeserializeOwned + Send {
    fn into_params(value: Option<Value>) -> Result<Self>;
}
```

Additional supported patterns:

- `IntoDefaultRpcParams`
  - if implemented, missing params resolve to `Default::default()`
- `IntoParams for Option<D>`
  - supports optional typed params when the wrapped type itself implements `IntoParams`
- `IntoParams for serde_json::Value`
  - permits raw JSON access when desired

The subsystem is intentionally narrow:

- only one logical params input is supported per handler
- params handling is explicit and type-driven
- missing params and parse failures are represented with dedicated library errors

### Error model

The crate-level `Error` type is the library error boundary for routing, param conversion, resource extraction, and handler invocation adaptation.

Current categories include:

```rust
pub enum Error {
    ParamsParsing(serde_json::Error),
    ParamsMissingButRequested,
    MethodUnknown,
    FromResources(FromResourcesError),
    HandlerResultSerialize(serde_json::Error),
    Handler(HandlerError),
}
```

Error responsibilities:

- provide a single library-level error surface
- preserve important error category boundaries
- allow application handler errors to flow through a wrapper type without forcing a single application error enum
- remain serializable where response shaping requires it

`HandlerError` is the bridge between application-defined handler error types and the library result model. User handler errors are converted into `HandlerError`, then wrapped as `Error::Handler(HandlerError)` for call-level reporting.

### Request and response boundary

The router operates on typed JSON-RPC request and response constructs.

Request boundary:

- `RpcRequest` contains JSON-RPC call data such as `id`, `method`, and `params`
- `RpcId` models valid JSON-RPC id forms
- request parsing and validation occur before handler dispatch

Response boundary:

- router execution returns `CallResult`
- success and error paths preserve request context
- success values are stored as serialized `serde_json::Value`
- errors are represented with the crate `Error` type

This keeps transport-agnostic code separate from routing semantics while still exposing enough structure for upstream integration layers to build protocol responses.

### Derive macro role

The derive macros support the core traits but do not redefine the architecture.

They generate trait implementations for:

- `RpcResource` -> `FromResources`
- `RpcParams` -> `IntoParams`
- `RpcHandlerError` -> `IntoHandlerError`

Implementation must continue to work equivalently when users write the trait impls manually.

## Design Considerations

- The router uses immutable shared state so it can be cloned and reused safely across async execution contexts.
- Dynamic dispatch is used at the route table level so handlers with different Rust function signatures can coexist in one router.
- Resource extraction is type-based rather than name-based, which keeps handler signatures idiomatic and avoids stringly typed dependency wiring.
- Resource overlays were chosen so per-call customization can happen without mutating or rebuilding router base state.
- Params are limited to a single optional JSON-RPC value input because that matches the protocol shape and keeps handler adaptation simpler.
- `serde_json::Value` is the normalized boundary for both incoming params and outgoing successful values, which keeps the router transport-agnostic and serialization-friendly.
- The error model separates parse, lookup, extraction, serialization, and handler-originated failures so callers can distinguish routing failures from application failures.
- Application handler errors are boxed behind `HandlerError` instead of requiring one global application error enum, which preserves flexibility for library consumers.
- The crate root re-exports a flattened API surface so common usage remains concise, while implementation details stay separated by module.
- Proc macro derives are treated as ergonomic helpers rather than required infrastructure, which keeps the core design trait-oriented and explicit.
