# Go SDK Reference

## Setup

### Restate Server

Install the Restate Server via one of:

```shell
# Docker (recommended)
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
go get github.com/restatedev/sdk-go
```

### Minimal scaffold

```go
package main

import (
  "context"
  "fmt"
  "log"

  restate "github.com/restatedev/sdk-go"
  server "github.com/restatedev/sdk-go/server"
)

type MyService struct{}

func (MyService) MyHandler(ctx restate.Context, greeting string) (string, error) {
  return fmt.Sprintf("%s!", greeting), nil
}

func main() {
  if err := server.NewRestate().
    Bind(restate.Reflect(MyService{})).
    Start(context.Background(), "0.0.0.0:9080"); err != nil {
    log.Fatal(err)
  }
}
```

### Register and invoke

Start the service, then register and invoke:

```shell
# Register the service with Restate Server
restate deployments register http://localhost:9080

# Invoke the handler
curl localhost:8080/MyService/MyHandler --json '"World"'
```

---

## Core Concepts

- Restate provides durable execution: if a handler crashes or the process restarts, Restate replays the handler from the last completed step, not from scratch.
- All handlers receive a Context object (`ctx`) as their first argument. Use restate actions with the ctx for all I/O and side effects.
- Handlers take one optional JSON-serializable input parameter and return one JSON-serializable output.

---

## Service types

Define handlers as exported methods on a struct. `restate.Reflect(MyService{})` discovers handlers via reflection. The struct name becomes the service name.

### Service (stateless)

See minimal scaffold above.

### Virtual Object (stateful, keyed)

```go
package main

import (
  "context"
  "fmt"
  "log"

  restate "github.com/restatedev/sdk-go"
  server "github.com/restatedev/sdk-go/server"
)

type MyObject struct{}

func (MyObject) MyHandler(ctx restate.ObjectContext, greeting string) (string, error) {
  return fmt.Sprintf("%s %s!", greeting, restate.Key(ctx)), nil
}

func (MyObject) MyConcurrentHandler(ctx restate.ObjectSharedContext, greeting string) (string, error) {
  return fmt.Sprintf("%s %s!", greeting, restate.Key(ctx)), nil
}

func main() {
  if err := server.NewRestate().
    Bind(restate.Reflect(MyObject{})).
    Start(context.Background(), "0.0.0.0:9080"); err != nil {
    log.Fatal(err)
  }
}
```

- Exclusive handlers (default): receive `restate.ObjectContext`. One invocation at a time per key.
- Shared handlers: receive `restate.ObjectSharedContext`. Concurrent, read-only.
- Access the key via `restate.Key(ctx)`.

### Workflow (exactly-once per ID)

```go
package myworkflow

import (
  "context"
  restate "github.com/restatedev/sdk-go"
  "github.com/restatedev/sdk-go/server"
  "log/slog"
  "os"
)

type MyWorkflow struct{}

func (MyWorkflow) Run(ctx restate.WorkflowContext, req string) (string, error) {
  // implement the workflow logic here
  return "success", nil
}

func (MyWorkflow) InteractWithWorkflow(ctx restate.WorkflowSharedContext) error {
  // implement interaction logic here
  // e.g. resolve a promise that the workflow is waiting on
  return nil
}

func main() {
  server := server.NewRestate().
    Bind(restate.Reflect(MyWorkflow{}))

  if err := server.Start(context.Background(), ":9080"); err != nil {
    slog.Error("application exited unexpectedly", "err", err.Error())
    os.Exit(1)
  }
}
```

- `Run` executes exactly once per workflow ID. Uses `restate.WorkflowContext`.
- Additional handlers use `restate.WorkflowSharedContext`. Run concurrently.
- Access the workflow ID via `restate.Key(ctx)`.

---

## State

Never use global variables for state -- it is not durable across restarts. Use Restate actions instead. Available in Virtual Objects and Workflows only.

```go
myString := "my-default"
if s, err := restate.Get[*string](ctx, "my-string-key"); err != nil {
  return err
} else if s != nil {
  myString = *s
}

count, err := restate.Get[int](ctx, "count")
if err != nil {
  return err
}

// Set state
restate.Set(ctx, "my-key", "my-new-value")
restate.Set(ctx, "count", count+1)

// Clear state
restate.Clear(ctx, "my-key")
restate.ClearAll(ctx)
stateKeys, err := restate.Keys(ctx)
```

Use `restate.Get[*string]` (pointer) when distinguishing "not set" (nil) from "set to zero value" matters. Use `restate.Get[int]` (value) when zero value is acceptable.

---

## Service Communication

### Request-response calls

```go
// Call a Service
svcResponse, err := restate.Service[string](ctx, "MyService", "MyHandler").
  Request(request)
if err != nil {
  return err
}

// Call a Virtual Object
objResponse, err := restate.Object[string](ctx, "MyObject", objectKey, "MyHandler").
  Request(request)
if err != nil {
  return err
}

// Call a Workflow
wfResponse, err := restate.Workflow[string](ctx, "MyWorkflow", workflowId, "Run").
  Request(request)
if err != nil {
  return err
}
```

### One-way messages (fire-and-forget)

```go
// Send to service
restate.ServiceSend(ctx, "MyService", "MyHandler").Send(request)

// Send to virtual object
restate.ObjectSend(ctx, "MyObject", objectKey, "MyHandler").Send(request)

// Send to workflow
restate.WorkflowSend(ctx, "MyWorkflow", workflowId, "Run").Send(request)
```

### Delayed messages

```go
restate.ServiceSend(ctx, "MyService", "MyHandler").Send(request, restate.WithDelay(5*time.Hour))
```

## Side effects (restate.Run)

Wrap all non-deterministic operations (API calls, DB writes, HTTP requests, file I/O) in `restate.Run` to journal their results.

```go
result, err := restate.Run(ctx, func(ctx restate.RunContext) (string, error) {
  return callExternalAPI(), nil
}, restate.WithName("Call to API"))
if err != nil {
  return err
}
```

### Async side effects (non-blocking)

```go
call1 := restate.RunAsync(ctx, func(ctx restate.RunContext) (string, error) {
  return callExternalAPI(), nil
})
user, err := call1.Result()
```

- No Restate context actions within `restate.Run()`.
- Always supply a name, used for observability and debugging.

---

## Deterministic helpers

Never use `rand.Int()`, `time.Now()`, or `uuid.New()` directly. These produce different values on replay.

```go
// Deterministic UUID
uuid := restate.UUID(ctx)

// Deterministic random numbers
randomInt := restate.Rand(ctx).Uint64()
randomFloat := restate.Rand(ctx).Float64()

// Use as a math/rand/v2 source
mathRandV2 := rand.New(restate.RandSource(ctx))

// time
now, err := restate.Run(ctx, func(ctx restate.RunContext) (time.Time, error) {
  return time.Now(), nil
})
```

---

## Timers (durable sleep)

Never use `time.Sleep()`. Use Restate's sleep instead for durable delays that survive crashes and restarts:

```go
err := restate.Sleep(ctx, 30*time.Second)
if err != nil {
  return err
}
```

No limit on duration, but long sleeps in exclusive handlers block other calls for that key.

### After (non-blocking timer future)

```go
sleepFuture := restate.After(ctx, 30*time.Second)
// ... do other work ...
if err := sleepFuture.Done(); err != nil {
  return err
}
```

---

## Awakeables

Pause a handler until an external system signals completion.

```go
awakeable := restate.Awakeable[string](ctx)
awakeableId := awakeable.Id()

// Send ID to external system
if _, err := restate.Run(ctx, func(ctx restate.RunContext) (string, error) {
  return requestHumanReview(name, awakeableId), nil
}); err != nil {
  return err
}

// Wait for result
review, err := awakeable.Result()
```

External systems can also resolve/reject via HTTP:
`curl localhost:8080/restate/awakeables/<id>/resolve --json '"Looks good!"'`

Or from another handler:

```go
restate.ResolveAwakeable(ctx, awakeableId, "Looks good!")
restate.RejectAwakeable(ctx, awakeableId, fmt.Errorf("Cannot be reviewed"))
```

---

## Durable promises

Cross-handler signaling within a Workflow. No ID management needed.

```go
// Wait for promise
promise := restate.Promise[string](ctx, "review")
review, err := promise.Result()
if err != nil {
  return err
}

// Resolve promise from another handler
err = restate.Promise[string](ctx, "review").Resolve(review)
if err != nil {
  return err
}
```

---

## Concurrency

CRITICAL: Use `restate.Wait` / `restate.WaitFirst`, NOT goroutines, channels, or Go `select` statements. Native concurrency primitives are not journaled and break deterministic replay.

### WaitFirst (race, first to complete)

```go
sleepFuture := restate.After(ctx, 30*time.Second)
callFuture := restate.Service[string](ctx, "MyService", "MyHandler").RequestFuture("hi")

fut, err := restate.WaitFirst(ctx, sleepFuture, callFuture)
```

### Wait (all, iterate over completions)

```go
callFuture1 := restate.Service[string](ctx, "MyService", "MyHandler").RequestFuture("hi")
callFuture2 := restate.Service[string](ctx, "MyService", "MyHandler").RequestFuture("hi again")

for fut, err := range restate.Wait(ctx, callFuture1, callFuture2) {
  if err != nil {
    return "", err
  }
  resp, _ := fut.(restate.ResponseFuture[string]).Response()
  return resp, nil
  // process response
}
```

---

## Invocation management

### Idempotency Keys

```go
invocationId := restate.ServiceSend(ctx, "MyService", "MyHandler").
  Send("Hi", restate.WithIdempotencyKey("my-key")).
  GetInvocationId()
```

This returns the invocation ID.

### Attach to a running invocation

```go
response, err := restate.AttachInvocation[string](ctx, invocationId).Response()
```

### Cancel an invocation

```go
restate.CancelInvocation(ctx, invocationId)
```

---

## Error handling

### Terminal errors (no retry)

```go
return restate.TerminalError(fmt.Errorf("Something went wrong."), 500)
```

`restate.TerminalError` is a function that wraps an error with an optional HTTP status code. Any other returned error will be retried infinitely with exponential backoff.

Catch terminal errors from `restate.Run` to handle permanent failures and execute compensations (see sagas guide).

---

## SDK clients (external invocations)

Call Restate services from outside of a Restate handler:

```go
//import restateingress "github.com/restatedev/sdk-go/ingress"
restateClient := restateingress.NewClient("http://localhost:8080")

// Request-response
result, err := restateingress.Service[string, string](
  restateClient, "MyService", "MyHandler").
  Request(context.Background(), "Hi")
if err != nil {
  // handle error
}

// One-way
restateingress.ServiceSend[string](
  restateClient, "MyService", "MyHandler").
  Send(context.Background(), "Hi")

// Delayed
restateingress.ServiceSend[string](
  restateClient, "MyService", "MyHandler").
  Send(context.Background(), "Hi", restate.WithDelay(1*time.Hour))
```

## Go-specific pitfalls (CRITICAL)

### 1. All Restate operations are package-level functions

Restate operations are NOT methods on ctx. Always use `restate.Run(ctx, ...)`, `restate.Get[T](ctx, ...)`, `restate.Set(ctx, ...)`, `restate.Sleep(ctx, ...)`, etc.

### 2. Always handle errors

Every Restate operation returns an error. Never use `_` to discard errors from Restate operations. Always check `err != nil`.

### 3. restate.Reflect() discovers handlers from struct methods

Only exported methods with valid signatures are discovered. Methods must have the correct context type as the first parameter. Check the package documentation for allowed signatures.

### 4. Do not use goroutines, channels, or select for Restate operations

Use `restate.WaitFirst` / `restate.Wait` instead. Goroutines and channels are safe ONLY inside `restate.Run` blocks.

### 5. Use restate.Void for handlers with no return value

`restate.Void{}` serializes to nil bytes. Use for `restate.Run` blocks and calls with no meaningful return.

### 6. Code generation alternative

The Go SDK supports defining handlers and types via Protocol Buffers for stronger type safety.

---

## Testing

Package: `github.com/restatedev/sdk-go/testing` (Testcontainers-based)

Tests run against a real Restate Server in Docker. 

```go
import (
  "testing"

  restate "github.com/restatedev/sdk-go"
  restateingress "github.com/restatedev/sdk-go/ingress"
  restatetest "github.com/restatedev/sdk-go/testing"
  "github.com/stretchr/testify/require"
)

func TestWithTestcontainers(t *testing.T) {
  tEnv := restatetest.Start(t, restate.Reflect(Greeter{}))
  client := tEnv.Ingress()

  out, err := restateingress.Service[string, string](client, "Greeter", "Greet").
    Request(t.Context(), "World")
  require.NoError(t, err)
  require.Equal(t, "You said hi to World!", out)
}
```

Use tests also to catch non-determinism bugs that unit tests miss: if handler code is non-deterministic, replay produces different results and the test fails.
You can do this by setting the environment variable `RESTATE_WORKER__INVOKER__INACTIVITY_TIMEOUT=0m` for the Restate Server.

---

## Further resources

- For detailed API: use the bundled restate-docs MCP server or GoDoc (https://pkg.go.dev/github.com/restatedev/sdk-go)
- Examples: https://github.com/restatedev/examples
- AI agent examples: https://github.com/restatedev/ai-examples
