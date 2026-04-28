# Python SDK Reference

## Setup

### Restate Server

Install the Restate Server via one of:

```shell
# Docker (recommended for Python projects)
docker run --name restate_dev --rm -p 8080:8080 -p 9070:9070 -p 9071:9071 \
  docker.io/restatedev/restate:latest

# Homebrew
brew install restatedev/tap/restate-server
restate-server

# npx (if Node.js is available)
npx @restatedev/restate-server
```

### Install Restate CLI

```bash
brew install restatedev/tap/restate
```

Or Docker:
```bash
docker run -it docker.restate.dev/restatedev/restate-cli:latest invocations ls
```

### SDK installation

```shell
pip install restate-sdk
# or with uv
uv add restate-sdk
```

Optional extras:

| Extra | Install command | Purpose |
|-------|----------------|---------|
| Pydantic support | `pip install restate-sdk[serde]` | Pydantic model serialization |
| Testing harness | `pip install restate-sdk[harness]` | Local testing without server |

### Minimal scaffold

```python
import restate

my_service = restate.Service("MyService")


@my_service.handler("myHandler")
async def my_handler(ctx: restate.Context, greeting: str) -> str:
    return f"{greeting}!"


app = restate.app([my_service])
```

### Register and invoke

Start the service with `uv run app.py`, then register and invoke:

```shell
restate deployments register http://localhost:9080
curl localhost:8080/MyService/myHandler --json '"World"'
```

---

## Core Concepts

- Restate provides durable execution: if a handler crashes or the process restarts, Restate replays the handler from the last completed step, not from scratch.
- All handlers receive a Context object (`ctx`) as their first argument. Use ctx methods for all I/O and side effects.
- Handlers take one optional JSON-serializable input parameter and return one JSON-serializable output.

---

## Service types

### Service (stateless)

See minimal scaffold above.

### Virtual Object (stateful, keyed)

```python
import restate

my_object = restate.VirtualObject("MyVirtualObject")


@my_object.handler("myHandler")
async def my_handler(ctx: restate.ObjectContext, greeting: str) -> str:
    return f"{greeting} {ctx.key()}!"


@my_object.handler(kind="shared")
async def my_concurrent_handler(ctx: restate.ObjectSharedContext, greeting: str) -> str:
    return f"{greeting} {ctx.key()}!"


app = restate.app([my_object])
```

- Exclusive handlers (default): one invocation at a time per key. Use for writes. Receive `ObjectContext`.
- Shared handlers (`kind="shared"`): concurrent, read-only. Receive `ObjectSharedContext`.
- Access the key via `ctx.key()`.

### Workflow

```python
import restate

my_workflow = restate.Workflow("MyWorkflow")


@my_workflow.main()
async def run(ctx: restate.WorkflowContext, req: str) -> str:
    # ... implement workflow logic here ---
    return "success"


@my_workflow.handler()
async def interact_with_workflow(ctx: restate.WorkflowSharedContext, req: str):
    # ... implement interaction logic here ...
    return


app = restate.app([my_workflow])
```

- `run` executes exactly once per workflow ID. Calling `run` again with the same ID attaches to the existing execution. Uses `WorkflowContext`.
- Other handlers can be called concurrently while `run` is in progress. Use them to send signals or read state. Use `WorkflowSharedContext`.

---

## State

Never use global variables for state -- it is not durable across restarts. Use `ctx.get`/`ctx.set` instead:

```python
count = await ctx.get("count", type_hint=int) or 0
ctx.set("count", count + 1)
ctx.clear("count")
ctx.clear_all()
keys = ctx.state_keys()
```

---

## Service Communication

### Request-response calls

```python
response = await ctx.service_call(my_handler, "Hi")
response2 = await ctx.object_call(my_object_handler, key="object-key", arg="Hi")
response3 = await ctx.workflow_call(run, "wf-id", arg="Hi")
```

### One-way messages (fire-and-forget)

```python
ctx.service_send(my_handler, "Hi")
ctx.object_send(my_object_handler, key="object-key", arg="Hi")
ctx.workflow_send(run, "wf-id", arg="Hi")
```

### Delayed messages

```python
ctx.service_send(my_handler, "Hi", send_delay=timedelta(hours=5))
```

### Generic calls (String-Based Service/Method Names)

Use when the service definition is not available at compile time:

```python
response_bytes = await ctx.generic_call(
    "MyObject", "my_handler", key="Mary", arg=json.dumps("Hi").encode("utf-8")
)
```

## Side effects (ctx.run)

Wrap all non-deterministic operations (API calls, DB writes, HTTP requests, file I/O) in `ctx.run` to journal their results.

```python
# Wrap non-deterministic code in ctx.run
result = await ctx.run(
    "my-side-effect", lambda: call_external_api("weather", "123")
)

# Or with typed version for better type safety
result = await ctx.run_typed(
    "my-side-effect", call_external_api, query="weather", some_id="123"
)
```

- The first argument is a label used for observability and debugging.
- The return value must be JSON-serializable with the json library or as Pydantic model (or use a custom serde).
- No Restate context actions within `ctx.run()`.

---

## Deterministic helpers

Never use `random.random()`, `time.time()`, `datetime.now()`, or `uuid.uuid4()` directly. These produce different values on replay. Instead:

```python
my_uuid = ctx.uuid()
ctx.random().random()
current_time = await ctx.time()
```

---

## Timers (durable sleep)

Never use `asyncio.sleep` or `time.sleep`. Use `ctx.sleep` for durable delays that survive crashes and restarts:

```python
# Sleep
await ctx.sleep(timedelta(seconds=30))

# Schedule delayed call (different from sleep + send)
ctx.service_send(my_handler, "Hi", send_delay=timedelta(hours=5))
```

No limit on duration, but long sleeps in exclusive handlers block other calls for that key.

---

## Awakeables

Pause a handler until an external system signals completion.

```python
# Create awakeable
awakeable_id, promise = ctx.awakeable(type_hint=str)

# Send ID to external system
await ctx.run_typed(
    "request_human_review",
    request_human_review,
    name=name,
    awakeable_id=awakeable_id,
)

# Wait for result
review = await promise
```

External systems can also resolve/reject via HTTP:
`curl localhost:8080/restate/awakeables/<id>/resolve --json '"Looks good!"'`

Or from another handler:

```python
ctx.resolve_awakeable(awakeable_id, "Looks good!")
ctx.reject_awakeable(awakeable_id, "Cannot be reviewed")
```

---

## Durable promises

Cross-handler signaling within a Workflow. No ID management needed.

```python
# Wait for promise
review = await ctx.promise("review", type_hint=str).value()

# Resolve promise
await ctx.promise("review", type_hint=str).resolve("approval")
```

---

## Concurrency

Never use `asyncio.gather` / `asyncio.wait`. Native asyncio combinators are not journaled and break deterministic replay. Use Restate Promises combinators instead.

### Gather (wait for all)

```python
# ❌ BAD
results1 = await asyncio.gather(call1, call2)

# ✅ GOOD
claude_call = ctx.service_call(ask_openai, "What is the weather?")
openai_call = ctx.service_call(ask_claude, "What is the weather?")
results2 = await restate.gather(claude_call, openai_call)
```

### Select (race, first to complete)

```python
# ❌ BAD
result1 = await asyncio.wait([call1, call2], return_when=asyncio.FIRST_COMPLETED)  # type: ignore[type-var]

# ✅ GOOD
confirmation = ctx.awakeable(type_hint=str)
match await restate.select(
    confirmation=confirmation[1], timeout=ctx.sleep(timedelta(days=1))
):
    case ["confirmation", result]:
        print("Got confirmation:", result)
    case ["timeout", _]:
        raise restate.TerminalError("Timeout!")
```

### Wait completed (completed + pending split)

```python
done, pending = await restate.wait_completed(call1, call2)
results = [await f for f in done]
# Cancel pending if needed
```

### As completed (process in completion order)

```python
async for future in restate.as_completed(call1, call2):
    print(await future)
```

---

## Invocation management

### Idempotency Keys

```python
# Send a request, get the invocation id
handle = ctx.service_send(
    my_handler, arg="Hi", idempotency_key="my-idempotency-key"
)
```

### Attach to a Running Invocation

```python
invocation_id = await handle.invocation_id()
result = await ctx.attach_invocation(invocation_id)
```

### Cancel an Invocation

```python
ctx.cancel_invocation(invocation_id)
```

---

## Serialization

### Default

JSON serialization via Python's `json` library. Primitive types, `TypedDict`, and simple dataclasses work out of the box.

### Pydantic models

Install `restate-sdk[serde]`. Then use Pydantic models directly as handler input/output and in state operations:

```python
import restate
from pydantic import BaseModel
from restate.serde import Serde


class Greeting(BaseModel):
    name: str


class GreetingResponse(BaseModel):
    result: str


greeter = restate.Service("Greeter")


@greeter.handler()
async def greet(ctx: restate.Context, greeting: Greeting) -> GreetingResponse:
    return GreetingResponse(result=f"You said hi to {greeting.name}!")
```

Pydantic models also work with `ctx.get`, `ctx.awakeable`, and `ctx.run_typed` via the `type_hint` parameter.

---

## Error handling

### Terminal errors (no retry)

Raise `TerminalError` to stop retries and propagate failure permanently:

```python
from restate import TerminalError

raise TerminalError("Invalid input - will not retry")
```

All other exceptions are retried with exponential backoff by default, and eventually paused.

Catch `TerminalError` from `ctx.run` to handle permanent failures and execute compensations (see sagas).

---

## Python-specific pitfalls (CRITICAL)

### 1. NEVER use bare `except:` or `except Exception:`

Bare exception handlers catch Restate SDK internal exceptions (suspension signals, terminal errors), causing silent failures and lost journal entries.

```python
# Bad: bare except swallows Restate's internal suspension signals
try:
    result = await ctx.run("api-call", call_external_api)
except:
    result = None

# Good: catch the specific exception you expect
try:
    result = await ctx.run("api-call", call_external_api)
except TimeoutError as e:
    raise TerminalError(f"API timeout: {e}")
```

### 2. Use Restate concurrency combinators, not asyncio

`asyncio.gather`, `asyncio.wait`, and `asyncio.as_completed` are not journaled and break deterministic replay. Use Restate's Promise combinators instead (see the Concurrency section above).

### 3. Use `ctx.run_typed` for better type safety

Prefer `ctx.run_typed("label", fn, **kwargs)` over `ctx.run("label", lambda: fn(...))` for anything more complex than a trivial lambda: the typed variant infers the return type from the function signature and works cleanly with Pydantic models.

### 4. No native random, time, or UUID

`random.random()`, `time.time()`, `datetime.now()`, and `uuid.uuid4()` produce different values on replay. Use Restate's deterministic helpers instead (see the Deterministic helpers section above).

### 5. No ctx operations inside ctx.run blocks

Read state before `ctx.run`, then use the value inside. No `ctx.get`, `ctx.set`, `ctx.sleep`, service calls, or nested `ctx.run` inside run blocks.

### 6. Return values from ctx.run must be JSON-serializable

Use Pydantic models or custom serializers for complex types.

---

## Testing

Install: `pip install restate-sdk[harness]`

Tests run against a real Restate Server in Docker via Testcontainers. 

```python
import restate

from ..my_service import app

with restate.test_harness(app, always_replay=True) as harness:
    client = harness.ingress_client()

    # Invoke a service handler
    response = client.post("/MyService/myHandler", json="Hello")
    assert response.json() == "Hello!"

    # Invoke a Virtual Object handler
    response = client.post("/MyObject/myKey/myHandler", json="Hello")
    assert response.json() == "Hello myKey!"
```

Use tests also to catch non-determinism bugs that unit tests miss: if handler code is non-deterministic, replay produces different results and the test fails.

---

## Further resources

- For detailed API: use the bundled restate-docs MCP server or Python SDK documentation
- Examples: https://github.com/restatedev/examples
- AI agent examples: https://github.com/restatedev/ai-examples
