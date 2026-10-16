# Flow Control, Scopes, and Limit Keys

Use this reference when an application needs concurrency limits, multi-tenant fairness, downstream protection, AI cost controls, scopes, or limit keys.

## Availability and status

Flow control is enabled by default since Restate Server 1.8. Its first capability is scope-based concurrency limiting. On Restate Server 1.7, it is an experimental opt-in feature.

Minimum SDK versions for the APIs in this reference:

| SDK | Minimum version |
| --- | --- |
| TypeScript | 1.15 |
| Python | 1.0 |
| Java / Kotlin | 2.9 |
| Go | 1.0 |

Before using these APIs, verify the project's Restate Server and SDK versions. Do not silently upgrade an existing project across an SDK major version.

## Enable flow control

On Restate Server 1.8 or later, flow control is enabled by default.

On Restate Server 1.7, enable protocol v7 and virtual queues:

```toml restate.toml
experimental-enable-protocol-v7 = true
experimental-enable-vqueues = true

# Required only for scoped calls to Virtual Objects
experimental-enable-scoped-virtual-objects = true
```

Environment variable equivalents:

```shell
RESTATE_EXPERIMENTAL_ENABLE_PROTOCOL_V7=true
RESTATE_EXPERIMENTAL_ENABLE_VQUEUES=true
RESTATE_EXPERIMENTAL_ENABLE_SCOPED_VIRTUAL_OBJECTS=true
```

Restate Server 1.7.3 or later automatically migrates an existing cluster's invocations to vqueues when you enable flow control. With Restate Server 1.7.0 through 1.7.2, enable flow control only on a fresh cluster with no in-flight invocations.

## Choose a scope

A scope is both a concurrency namespace and part of resource identity.

Good scope candidates include a tenant, organization, downstream dependency, or class of expensive work. Choose a scope with enough cardinality to distribute scheduling across partitions: every invocation of a scope, with its journal, state and locks, is processed by one partition, so one partition processor bounds a scope's throughput.

A scope changes identity in these ways:

- The same idempotency key deduplicates independently in each scope.
- The same Virtual Object key in two scopes addresses two separate objects with separate state and queues.
- The same Workflow key in two scopes addresses two separate workflow instances.

A call without a scope reaches the unscoped Virtual Object. Scope an object when it belongs to the scope, such as a per-tenant cart, and its work should count against that scope's budget. Keep it unscoped when the key cardinality is high and no flow control is needed. Do not mix scoped and unscoped calls to the same key unless separate objects are intended.

A scope must be 1 to 36 characters and contain only `a-z`, `A-Z`, `0-9`, `_`, `.`, or `-`.

## Add limit keys

A limit key subdivides concurrency within a scope into one or two hierarchical levels such as `team` or `team/user`.

An invocation with `team/user` consumes capacity from the scope, `team`, and `team/user` limits at the same time. The strictest matching limit determines whether it can run.

Limit keys affect concurrency only. They are not part of invocation, Virtual Object, or Workflow identity. A limit key always requires a scope.

Calls to one Virtual Object instance with the same limit key run in arrival order. Across different limit keys the order is not guaranteed, because each limit key has its own queue.

## Configure concurrency rules

Rules apply cluster-wide and can match a scope plus zero, one, or two limit-key levels:

```bash
# Default limit for each scope
restate rules set "*" --concurrency 1000 --description "scope default"

# Tighter limit for one scope
restate rules set "checkout" --concurrency 50

# Limits for each first-level and second-level key under checkout
restate rules set "checkout/*" --concurrency 5
restate rules set "checkout/*/*" --concurrency 2

restate rules list --extra
```

The wildcard rule is not a global shared pool. `*` gives every matching scope its own limit.

## Call from a handler

### TypeScript

```ts
// Route a call into a named scope
const svcResponse = await ctx
  .scope("tenant-123")
  .serviceClient(myService)
  .myHandler("Hi");

// Add a limit key for hierarchical concurrency limits within the scope
const wfResponse = await ctx
  .scope("tenant-123")
  .workflowClient(myWorkflow, "wf-id")
  .run("Hi", restate.rpc.opts({ limitKey: "premium/user42" }));

const objResponse = await ctx
  .scope("tenant-123")
  .objectClient(myObject, "Mary")
  .myHandler("Hi");

// Fire-and-forget sends can be scoped too
ctx.scope("tenant-123").serviceSendClient(myService).myHandler("Hi");
```

### Python

```python
# Route a call into a named scope
response = await ctx.scope("tenant-123").service_call(my_service_handler, arg="Hi")

# Add a limit key for hierarchical concurrency limits within the scope
response = await ctx.scope("tenant-123").workflow_call(
    run, key="my_workflow_id", arg="Hi", limit_key="premium/user42"
)

# Fire-and-forget sends can be scoped too
ctx.scope("tenant-123").object_send(my_object_handler, key="Mary", arg="Hi")
```

### Java

```java
// Route a call into a named scope
String svcResponse = Restate.scope("tenant-123").service(MyService.class).myHandler(request);

// Add a limit key for hierarchical concurrency limits within the scope
String wfResponse =
    Restate.scope("tenant-123")
        .workflowHandle(MyWorkflow.class, workflowId)
        .call(MyWorkflow::run, request, InvocationOptions.limitKey("premium/user42"))
        .await();

String objResponse =
    Restate.scope("tenant-123").virtualObject(MyObject.class, objectKey).myHandler(request);
```

### Go

```go
// Route a call into a named scope with WithScope (a client option)
svcResponse, err := restate.Service[string](ctx, "MyService", "MyHandler", restate.WithScope("tenant-123")).
  Request("Hi")

// Add a limit key for hierarchical concurrency limits within the scope
wfResponse, err := restate.Workflow[bool](ctx, "MyWorkflow", "my-workflow-id", "Run", restate.WithScope("tenant-123")).
  Request("Hi", restate.WithLimitKey("premium/user42"))

objResponse, err := restate.Object[string](ctx, "MyObject", "Mary", "MyHandler", restate.WithScope("tenant-123")).
  Request("Hi")

// Fire-and-forget sends can be scoped too
restate.ServiceSend(ctx, "MyService", "MyHandler", restate.WithScope("tenant-123")).Send("Hi")
```

## Call over HTTP

Use the scoped ingress endpoint and pass an optional limit key in the query or `x-restate-limit-key` header:

```shell
curl "localhost:8080/restate/scope/acme/call/Inference/run?limit-key=growth/alice" \
  --json '{"prompt":"..."}'
```

Calls through non-scoped endpoints do not consume scope-based concurrency limits.

## Verify the design

- Confirm scope cardinality is high enough to avoid hot partitions.
- Confirm that separating resource identity by scope is intentional.
- Confirm every limit key has a scope.
- On Restate Server 1.7, confirm scoped Virtual Object calls have the extra server flag enabled.
- Add rules before expecting scoped invocations to be limited.
- Test with a Restate 1.8 or later server image and the project's actual SDK version.

Full documentation: https://docs.restate.dev/services/flow-control
