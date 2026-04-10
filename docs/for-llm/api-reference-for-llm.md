# rpc-router API Reference for LLM

## Core Request/Response Types

### RpcId
Enum for JSON-RPC 2.0 IDs.
- `String(Arc<str>)`
- `Number(i64)`
- `Null`
Constructors: `new_uuid_v4()`, `new_uuid_v7()`, `from_scheme(IdSchemeKind, IdSchemeEncoding)`.

### RpcRequest
Primary request structure for methods expecting a response.
- `id: RpcId`
- `method: String`
- `params: Option<serde_json::Value>`
Methods: `new(id, method, params)`, `from_value(Value)`, `from_value_with_checks(Value, RpcRequestCheckFlags)`.

Notes:
- `from_value` validates with `RpcRequestCheckFlags::ALL`.
- `from_value_with_checks` validates `jsonrpc: "2.0"` only when `VERSION` is set.
- `id` is required only when `ID` is set. If `ID` is not set, missing or invalid ids fall back to `RpcId::Null`.
- `params` is optional and is taken as-is from the request object if present.

### RpcNotification
Request structure for notifications (no response).
- `method: String`
- `params: Option<serde_json::Value>`
Methods: `from_value(Value)`.

### RpcResponse
Enum representing the final JSON-RPC response.
- `Success(RpcSuccessResponse)`
- `Error(RpcErrorResponse)`
Methods: `id()`, `is_success()`, `is_error()`, `into_parts()`.
Conversions: `From<CallResult>`, `From<CallSuccess>`, `From<CallError>`.

### RpcError
The JSON-RPC 2.0 Error Object.
- `code: i64`
- `message: String`
- `data: Option<Value>`
Predefined codes: `CODE_PARSE_ERROR (-32700)`, `CODE_INVALID_REQUEST (-32600)`, `CODE_METHOD_NOT_FOUND (-32601)`, `CODE_INVALID_PARAMS (-32602)`, `CODE_INTERNAL_ERROR (-32603)`.

## Router and Resources

### Router
The main executor. Usually wrapped in `Arc`.
- `call(RpcRequest)`: Exec with router's base resources.
- `call_with_resources(RpcRequest, Resources)`: Exec with additional overlaid resources.
- `call_route(id, method, params)`: Lower level call.
- `call_route_with_resources(id, method, params, Resources)`: Lower level call with overlaid resources.

Behavior notes:
- Resource lookup checks overlay resources first, then router base resources.
- `call_route` and `call_route_with_resources` default `id` to `RpcId::Null` when passed `None`.

### RouterBuilder
- `append(name, handler_fn)`: Generic add.
- `append_dyn(name, handler_fn.into_dyn())`: Type-erased add (recommended for large routers).
- `append_resource(val)`: Add base resource to all calls.
- `extend_resources(Option<ResourcesBuilder>)`: Extend base resources from an optional builder.
- `set_resources(ResourcesBuilder)`: Replace builder base resources.
- `extend(other_builder)`: Merge routes and resources.
- `build()`: Returns `Router`.

### Resources
Type-safe container for shared state.
- `Resources::builder().append(T).build()`
- `get<T>()`: Returns `Option<T>`. Looks in overlay then base.
- `is_empty()`: Returns true when both base and overlay resource stores are empty.

### ResourcesBuilder
- `append(T)`: Append a resource and return the builder.
- `append_mut(T)`: Append a resource without consuming the builder.
- `get<T>()`: Read a typed resource from the builder.
- `build()`: Produce `Resources`.

## Traits for Handlers

### FromResources
Used for resource injection arguments (0 to 8).
- Implement for types to be fetched from the `Resources` map.

### Handler
Trait implemented by supported async handler functions.
- `call(self, Resources, Option<Value>) -> Future<Output = Result<Value>>`
- `into_dyn()`: Convert the handler into `Box<dyn RpcHandlerWrapperTrait>`.

Key behavior:
- Handlers are async and produce `Result<Value>`.
- Handler arguments are normalized as injected `FromResources` values plus an optional final params value.
- Dynamic dispatch goes through `RpcHandlerWrapperTrait`.

### IntoParams
Used for the last argument of a handler function.
- Implement for structs to be used as `params`.
- Default impl uses `serde_json::from_value`.

### IntoHandlerError
Allows application errors to be converted into `HandlerError`.
- `HandlerError` holds the original error in an `Any` map.
- Retrieve original error with `handler_error.get::<T>()` or `remove::<T>()`.

## Macros and Derives

### Macros
- `router_builder![handlers: [...], resources: [...]]`
- `resources_builder![...]`

### Derives
- `#[derive(RpcParams)]`: Implements `IntoParams`.
- `#[derive(RpcResource)]`: Implements `FromResources`.
- `#[derive(RpcHandlerError)]`: Implements `IntoHandlerError` and `std::error::Error`.

## Error Types
- `rpc_router::Error`: Routing level errors (MethodUnknown, ParamsParsing, etc.).
- `CallError`: Contextual error containing `RpcId`, `method`, and `rpc_router::Error`.
- `RpcRequestParsingError`: Validation errors during request parsing.

## Notes for Current Public Surface

- `Router::builder()` returns `RouterBuilder::default()`.
- `Resources::builder()` returns `ResourcesBuilder::default()`.
- `Router` implements `FromResources`, so a router can itself be injected as a resource.
- `RpcResponse` supports `From<CallResult>`, `From<CallSuccess>`, and `From<CallError>`.
- `RpcRequest` also implements `TryFrom<Value>`, delegating to `RpcRequest::from_value`.
