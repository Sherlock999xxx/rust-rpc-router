# RPC Router - All APIs

This document provides a comprehensive overview of the current public APIs exposed by the `rpc-router` crate.

## Key Features

- Type-safe async handlers with typed resource injection and optional typed params.
- Immutable shared router and resource state designed for concurrent async use.
- JSON-RPC request, notification, id, and response types for protocol-facing code.
- Typed intermediate `CallResult` values that preserve request context before final response shaping.
- Flexible handler error adaptation through `HandlerError` and `IntoHandlerError`.
- Optional derive macros and builder macros for more ergonomic setup.

## Core Request and Response Types

### `RpcId`

Enum for JSON-RPC 2.0 IDs.

- `String(Arc<str>)`
- `Number(i64)`
- `Null`

Constructors include:

- `RpcId::from_scheme(kind: IdSchemeKind, enc: IdSchemeEncoding)`
- `RpcId::new_uuid_v4()`
- `RpcId::new_uuid_v7()`

Conversion helpers include:

- `From<String>`
- `From<&str>`
- `From<i64>`
- `From<i32>`
- `From<u32>`
- `to_value(&self) -> Value`
- `from_value(value: Value) -> Result<Self, RpcRequestParsingError>`

### `RpcRequest`

Primary request structure for methods expecting a response.

```rust
pub struct RpcRequest {
    pub id: RpcId,
    pub method: String,
    pub params: Option<serde_json::Value>,
}
```

Main constructors and parsers:

- `new(id, method, params)`
- `from_value(Value)`
- `from_value_with_checks(Value, RpcRequestCheckFlags)`
- `TryFrom<Value>`

Behavior notes:

- `from_value` validates with `RpcRequestCheckFlags::ALL`.
- `from_value_with_checks` validates `jsonrpc: "2.0"` only when `VERSION` is set.
- `id` is required only when `ID` is set.
- If `ID` is not set, missing or invalid ids fall back to `RpcId::Null`.
- `params` is optional and is taken as-is from the request object if present.

Example:

```rust
use rpc_router::{RpcId, RpcRequest, RpcRequestCheckFlags};
use serde_json::json;

let request_value = json!({
    "jsonrpc": "2.0",
    "method": "my_method",
    "params": [1, 2]
});

let request = RpcRequest::from_value_with_checks(
    request_value,
    RpcRequestCheckFlags::VERSION,
)?;

assert_eq!(request.id, RpcId::Null);
assert_eq!(request.method, "my_method");
# Ok::<(), rpc_router::RpcRequestParsingError>(())
```

### `RpcNotification`

Request structure for notifications that do not produce a response.

```rust
pub struct RpcNotification {
    pub method: String,
    pub params: Option<serde_json::Value>,
}
```

Main parsers:

- `from_value(Value)`
- `TryFrom<Value>`

Behavior notes:

- Validates `"jsonrpc": "2.0"`.
- Requires a valid `method`.
- Validates `params` type when present.
- Rejects payloads that contain an `id`.

### `RpcResponse`

Final JSON-RPC response enum.

```rust
pub enum RpcResponse {
    Success(RpcSuccessResponse),
    Error(RpcErrorResponse),
}
```

Main methods:

- `id()`
- `is_success()`
- `is_error()`
- `into_parts()`

Conversions:

- `From<CallResult>`
- `From<CallSuccess>`
- `From<CallError>`

### `RpcError`

JSON-RPC 2.0 Error Object.

```rust
pub struct RpcError {
    pub code: i64,
    pub message: String,
    pub data: Option<serde_json::Value>,
}
```

Predefined codes:

- `CODE_PARSE_ERROR`
- `CODE_INVALID_REQUEST`
- `CODE_METHOD_NOT_FOUND`
- `CODE_INVALID_PARAMS`
- `CODE_INTERNAL_ERROR`

## Router and Resources

### `Router`

The main executor. Usually wrapped in an `Arc` when shared externally, though the type itself is already cheap to clone.

Primary execution methods:

- `call(RpcRequest)`
- `call_with_resources(RpcRequest, Resources)`
- `call_route(id, method, params)`
- `call_route_with_resources(id, method, params, Resources)`

Terminology:

- Base resources are the router-wide resources attached to the `Router` itself, usually through `RouterBuilder`.
- Overlay resources are the call-local `Resources` values passed to `call_with_resources` or `call_route_with_resources`.
- When the same resource type exists in both places, overlay lookup takes precedence over base lookup.

Behavior notes:

- Resource lookup checks overlay resources first, then router base resources.
- `call_route` and `call_route_with_resources` default `id` to `RpcId::Null` when passed `None`.

Example:

```rust
use rpc_router::{Resources, Router, RouterBuilder, RpcId, RpcRequest, RpcResource};
use serde_json::json;

#[derive(Clone, RpcResource)]
struct AppName(String);

#[derive(Clone, RpcResource)]
struct RequestScopedUser(String);

let router = RouterBuilder::default()
    .append_resource(AppName("rpc-router".to_string()))
    .build();

let overlay_resources = Resources::builder()
    .append(RequestScopedUser("alice".to_string()))
    .build();

let request = RpcRequest::new(
    RpcId::from(1_i64),
    "whoami",
    Some(json!({"verbose": true})),
);

let _call_result = router
    .call_with_resources(request, overlay_resources)
    .await;
```

In this pattern, the router keeps stable base resources, while each call can supply temporary overlay resources built with `Resources::builder().append(...).build()`.

### `RouterBuilder`

Builder for configuring routes and base resources.

- `append(name, handler_fn)`
- `append_dyn(name, handler_fn.into_dyn())`
- `append_resource(val)`
- `extend_resources(Option<ResourcesBuilder>)`
- `set_resources(ResourcesBuilder)`
- `extend(other_builder)`
- `build()`

Notes:

- `append` is ergonomic and generic.
- `append_dyn` is useful for type-erased route registration and reducing monomorphization pressure in larger routers.

### `Resources`

Type-safe container for shared state.

- `Resources::builder().append(T).build()`
- `get<T>() -> Option<T>`
- `is_empty() -> bool`

Lookup checks overlay resources first, then base resources.

Terminology:

- A base resource is stored in the router and is available to every call.
- An overlay resource is created for a specific call and temporarily layered on top of the base resources.
- If both layers contain the same type, `get<T>()` returns the overlay value.

Example:

```rust
use rpc_router::{Resources, RpcResource};

#[derive(Clone, RpcResource)]
struct AppVersion(&'static str);

#[derive(Clone, RpcResource)]
struct RequestId(&'static str);

let base_resources = Resources::builder()
    .append(AppVersion("1.0.0"))
    .build();

let overlay_resources = Resources::builder()
    .append(RequestId("req-123"))
    .build();

let _ = (base_resources, overlay_resources);
```

### `ResourcesBuilder`

Mutable builder for constructing `Resources`.

- `append(T)`
- `append_mut(T)`
- `get<T>()`
- `build()`

Common overlay construction pattern:

```rust
use rpc_router::Resources;

let call_resources = Resources::builder()
    .append("request-local value".to_string())
    .build();
```

## Router Call Results

### `CallResult`

Type alias for:

```rust
pub type CallResult = Result<CallSuccess, CallError>;
```

This is the intermediate result returned by router execution before it is converted into a final `RpcResponse`.

### `CallSuccess`

Successful routed call result.

```rust
pub struct CallSuccess {
    pub id: RpcId,
    pub method: String,
    pub value: serde_json::Value,
}
```

### `CallError`

Failed routed call result.

```rust
pub struct CallError {
    pub id: RpcId,
    pub method: String,
    pub error: rpc_router::Error,
}
```

This preserves the original request context together with the router-level error.

## Traits for Handlers

### `FromResources`

Used for resource injection arguments.

- Implement this for types that should be fetched from the `Resources` map.
- Typical bound expectations are `Clone + Send + Sync + 'static`.
- There is also a blanket impl for `Option<T>` where `T: FromResources`.

### `Handler`

Trait implemented by supported async handler functions.

Key behavior:

- Handlers are async and produce a normalized result.
- Handler arguments are normalized as injected `FromResources` values plus an optional final params value.
- Dynamic dispatch goes through `RpcHandlerWrapperTrait`.

Representative surface:

```rust
fn call(self, resources: Resources, params: Option<Value>) -> PinFutureValue;
fn into_dyn(self) -> Box<dyn RpcHandlerWrapperTrait>;
```

In practice, most applications use plain async functions and let the library implement `Handler` automatically.

### `IntoParams`

Used for the last argument of a handler function.

- Implement for structs or other types that should be deserialized from JSON-RPC `params`.
- The default implementation uses `serde_json::from_value`.
- Missing params return `Error::ParamsMissingButRequested`.

Supported patterns:

- Plain custom type implementing `IntoParams`
- `Option<T>` where `T: IntoParams`
- `serde_json::Value` for raw JSON access

### `IntoDefaultRpcParams`

Marker trait for param types that should fall back to `Default::default()` when params are missing or null.

This works together with `Default` to provide automatic `IntoParams` behavior for missing params.

### `IntoHandlerError`

Allows application errors to be converted into `HandlerError`.

- `HandlerError` stores the original error type in an erased form.
- Applications can recover the original type with `handler_error.get::<T>()` or `remove::<T>()`.

## Error Types

### `rpc_router::Error`

Routing-level error enum used inside `CallError`.

Representative variants include:

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

### `FromResourcesError`

Error for resource extraction failures.

```rust
pub enum FromResourcesError {
    ResourceNotFound(&'static str),
}
```

### `HandlerError`

Wrapper around errors returned by handler functions.

Useful methods:

- `HandlerError::new<T>(val)`
- `get<T>(&self) -> Option<&T>`
- `remove<T>(&mut self) -> Option<T>`
- `type_name(&self) -> &'static str`

## Macros and Derives

### Macros

- `router_builder![handlers: [...], resources: [...]]`
- `resources_builder![...]`

### Derives

- `#[derive(RpcParams)]`, implements `IntoParams`
- `#[derive(RpcResource)]`, implements `FromResources`
- `#[derive(RpcHandlerError)]`, implements `IntoHandlerError` and error boilerplate

## Request Handling Flow

1. Parse incoming JSON into `RpcRequest` or `RpcNotification`.
2. For requests, call `Router::call(...)` or `Router::call_with_resources(...)`.
3. Resolve the method name to a registered handler.
4. Resolve required resources from the effective `Resources`.
5. Convert optional JSON params into the handler params type.
6. Execute the async handler.
7. Normalize the result into `CallResult`.
8. Convert `CallResult` into `RpcResponse` when producing a JSON-RPC response payload.

## Usage Example

```rust
use rpc_router::{
    resources_builder, router_builder, HandlerResult, RpcParams, RpcRequest, RpcResource,
    RpcResponse,
};
use serde::{Deserialize, Serialize};
use serde_json::json;

#[derive(Clone, RpcResource)]
struct AppState {
    version: String,
}

#[derive(Clone, RpcResource)]
struct DbPool {
    url: String,
}

#[derive(Deserialize, Serialize, RpcParams)]
struct HelloParams {
    name: String,
}

async fn hello(state: AppState, params: HelloParams) -> HandlerResult<String> {
    Ok(format!("Hello {}, from app version {}!", params.name, state.version))
}

async fn get_db_url(db: DbPool) -> HandlerResult<String> {
    Ok(db.url.clone())
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let resources = resources_builder![
        AppState {
            version: "1.0".to_string()
        },
        DbPool {
            url: "dummy-db-url".to_string()
        }
    ]
    .build();

    let router = router_builder![
        handlers: [hello, get_db_url]
    ]
    .build();

    let request_json = json!({
        "jsonrpc": "2.0",
        "id": 1,
        "method": "hello",
        "params": {"name": "World"}
    });

    let rpc_request = RpcRequest::from_value(request_json)?;
    let call_result = router.call_with_resources(rpc_request, resources).await;
    let rpc_response = RpcResponse::from(call_result);

    println!("{}", serde_json::to_string_pretty(&rpc_response)?);

    Ok(())
}
```
