# TypeScript SDK Reference: API and Pitfalls

## Setup

### Install Restate Server

Ask the user for preferred installation method:

**npx (quick, no install):**
```bash
npx @restatedev/restate-server
```

**Homebrew:**
```bash
brew install restatedev/tap/restate-server
```

**Docker:**
```bash
docker run --name restate_dev --rm -p 8080:8080 -p 9070:9070 docker.io/restatedev/restate
```

### Install Restate CLI

```bash
brew install restatedev/tap/restate
```

Or via npx (no install):

```bash
npx @restatedev/restate 
```

Or Docker:
```bash
docker run -it docker.restate.dev/restatedev/restate-cli:latest invocations ls
```

### Install SDK

```bash
npm install @restatedev/restate-sdk
```

Optional packages:
- `@restatedev/restate-sdk-zod` -- Zod validation for handler input/output
- `@restatedev/restate-sdk-clients` -- invoke Restate handlers from external clients
- `@restatedev/restate-sdk-testcontainers` -- testing utilities

### Minimal Scaffold

```ts
import * as restate from "@restatedev/restate-sdk";

export const myService = restate.service({
  name: "MyService",
  handlers: {
    myHandler: async (ctx: restate.Context, greeting: string) => {
      return `${greeting}!`;
    },
  },
});

restate.serve({ services: [myService] });
```

### Register and Invoke

Start the service (e.g., `npx tsx src/app.ts`), then register and invoke:

```bash
restate deployments register http://localhost:9080
curl localhost:8080/MyService/greet --json '"World"'
```

---

## Core Concepts

- Restate provides durable execution: if a handler crashes or the process restarts, Restate replays the handler from the last completed step, not from scratch.
- All handlers receive a Context object (`ctx`) as their first argument. Use ctx methods for all I/O and side effects.
- Handlers take one optional JSON-serializable input parameter and return one JSON-serializable output.

---

## Service Types

### Service (Stateless)

See minimal scaffold above.

### Virtual Object (Stateful, Keyed)

```ts
import * as restate from "@restatedev/restate-sdk";

export const myObject = restate.object({
  name: "MyObject",
  handlers: {
    myHandler: async (ctx: restate.ObjectContext, greeting: string) => {
      return `${greeting} ${ctx.key}!`;
    },
    myConcurrentHandler: restate.handlers.object.shared(
      async (ctx: restate.ObjectSharedContext, greeting: string) => {
        return `${greeting} ${ctx.key}!`;
      }
    ),
  },
});

restate.serve({ services: [myObject] });
```

- **Exclusive handlers** (default): only one executes at a time per key. Use for writes. Receive `ObjectContext`.
- **Shared handlers** (`restate.handlers.object.shared`): run concurrently per key. Use for reads. Receive `ObjectSharedContext`.

### Workflow

```ts
import * as restate from "@restatedev/restate-sdk";

export const myWorkflow = restate.workflow({
  name: "MyWorkflow",
  handlers: {
    run: async (ctx: restate.WorkflowContext, req: string) => {
      // implement workflow logic here

      return "success";
    },

    interactWithWorkflow: async (ctx: restate.WorkflowSharedContext) => {
      // implement interaction logic here
      // e.g. resolve a promise that the workflow is waiting on
    },
  },
});

restate.serve({ services: [myWorkflow] });
```

- `run` executes exactly once per workflow ID. Calling `run` again with the same ID attaches to the existing execution. Uses `WorkflowContext`.
- Other handlers can be called concurrently while `run` is in progress. Use them to send signals or read state. Use `WorkflowSharedContext`.

---

## State Management

Never use global variables for state -- it is not durable across restarts. Use `ctx.get`/`ctx.set` instead:

```ts
const count = (await ctx.get<number>("count")) ?? 0;
ctx.set("count", count + 1);
ctx.clear("count");
ctx.clearAll();
const keys = await ctx.stateKeys();
```

---

## Service Communication

### Request-Response Calls

```ts
// Call a Service
const response = await ctx.serviceClient(myService).myHandler("Hi");

// Call a Virtual Object
const response2 = await ctx.objectClient(myObject, "key").myHandler("Hi");

// Call a Workflow
const response3 = await ctx.workflowClient(myWorkflow, "wf-id").run("Hi");
```

### One-Way Calls (Fire-and-Forget)

```ts
ctx.serviceSendClient(myService).myHandler("Hi");
ctx.objectSendClient(myObject, "key").myHandler("Hi");
ctx.workflowSendClient(myWorkflow, "wf-id").run("Hi");
```

### Delayed messages

```ts
ctx
  .serviceSendClient(myService)
  .myHandler("Hi", restate.rpc.sendOpts({ delay: { hours: 5 } }));
```

### Generic Calls (String-Based Service/Method Names)

Use when the target service type is not available at compile time:

```ts
const response = await ctx.genericCall({
  service: "MyObject",
  method: "myHandler",
  parameter: "Hi",
  key: "Mary", // drop this for Service calls
  inputSerde: restate.serde.json,
  outputSerde: restate.serde.json,
});
```

Generic send variant:

```ts
ctx.genericSend({
  service: "MyService",
  method: "myHandler",
  parameter: "Hi",
  inputSerde: restate.serde.json,
});
```

---

## Side Effects / ctx.run

Never call external APIs, databases, or non-deterministic functions directly in a handler. Wrap them in `ctx.run`:

```ts
const result = await ctx.run("my-side-effect", async () => {
  return await callExternalAPI();
});
```

- The first argument is a label used for observability and debugging.
- The return value must be JSON-serializable (or use a custom serde).
- No Restate context actions within `ctx.run()`.

---

## Deterministic Helpers

Never use `Math.random()`, `Date.now()`, or `new Date()` directly -- they break deterministic replay. Use ctx helpers instead:

```ts
const value = ctx.rand.random();
const uuid = ctx.rand.uuidv4();
const now = await ctx.date.now();
```

---

## Durable Timers

Never use `setTimeout`. Use `ctx.sleep` for durable delays that survive crashes and restarts:

```ts
await ctx.sleep({ seconds: 30 });
```

No limit on duration, but long sleeps in exclusive handlers block other calls for that key.

---

## Awakeables

Awakeables pause execution until an external system signals completion:

```ts
// Create awakeable
const { id, promise } = ctx.awakeable<string>();

// Send ID to external system
await ctx.run(() => requestHumanReview(name, id));

// Wait for result
const review = await promise;
```

External systems can also resolve/reject via HTTP:
`curl localhost:8080/restate/awakeables/<id>/resolve --json '"Looks good!"'`

Or from another handler:

```ts
// Resolve from another handler
ctx.resolveAwakeable(id, "Looks good!");

// Reject from another handler
ctx.rejectAwakeable(id, "Cannot be reviewed");
```

---

## Durable Promises (Workflows Only)

Durable promises allow communication between a workflow's `run` handler and its shared handlers:

```ts
// Wait for promise
const review = await ctx.promise<string>("review");

// Resolve promise
await ctx.promise<string>("review").resolve(review);
```

---

## Concurrency

Always use `RestatePromise` (imported from `@restatedev/restate-sdk`), NOT native `Promise`:

```ts
import { RestatePromise } from "@restatedev/restate-sdk";
```

### All (wait for all to complete)

```ts
// ❌ BAD
const results1 = await Promise.all([call1, call2]);

// ✅ GOOD
const claude = ctx.serviceClient(claudeAgent).ask("What is the weather?");
const openai = ctx.serviceClient(openAiAgent).ask("What is the weather?");
const results2 = await RestatePromise.all([claude, openai]);
```

### Race (first to settle)

```ts
// ❌ BAD
const result1 = await Promise.race([call1, call2]);

// ✅ GOOD
const firstToComplete = await RestatePromise.race([
  ctx.sleep({ milliseconds: 100 }),
  ctx.serviceClient(myService).myHandler("Hi"),
]);
```

### Any (first to succeed)

```ts
// ❌ BAD - using Promise.any (not journaled)
const result1 = await Promise.any([call1, call2]);

// ✅ GOOD
const result2 = await RestatePromise.any([
  ctx.run(() => callLLM("gpt-4", prompt)),
  ctx.run(() => callLLM("claude", prompt)),
]);
```

### AllSettled (wait for all, regardless of success/failure)

```ts
// ❌ BAD
const results1 = await Promise.allSettled([call1, call2]);

// ✅ GOOD
const results2 = await RestatePromise.allSettled([
  ctx.serviceClient(service1).call(),
  ctx.serviceClient(service2).call(),
]);

results2.forEach((result, i) => {
  if (result.status === "fulfilled") {
    console.log(`Call ${i} succeeded:`, result.value);
  } else {
    console.log(`Call ${i} failed:`, result.reason);
  }
});
```

---

## Invocation Management

### Idempotency Keys

```ts
const handle = ctx
  .serviceSendClient(myService)
  .myHandler("Hi", restate.rpc.sendOpts({ idempotencyKey: "my-key" }));
```

### Attach to a Running Invocation

```ts
const invocationId = await handle.invocationId;
const response = await ctx.attach(invocationId);
```

### Cancel an Invocation

```ts
ctx.cancel(invocationId);
```

---

## Serialization

### Default: JSON

All handler inputs/outputs and state values use JSON serialization by default.

### Zod Validation

Install `@restatedev/restate-sdk-zod`, then define schemas:

```ts
import * as restate from "@restatedev/restate-sdk";
import { z } from "zod";
import { serde } from "@restatedev/restate-sdk-zod";

const Greeting = z.object({
  name: z.string(),
});

const GreetingResponse = z.object({
  result: z.string(),
});

const greeter = restate.service({
  name: "Greeter",
  handlers: {
    greet: restate.handlers.handler(
      { input: serde.zod(Greeting), output: serde.zod(GreetingResponse) },
      async (ctx: restate.Context, { name }) => {
        return { result: `You said hi to ${name}!` };
      }
    ),
  },
});
```

### Binary Data

```ts
const myService = restate.service({
  name: "MyService",
  handlers: {
    myHandler: restate.handlers.handler(
      {
        // Set the input serde here
        input: restate.serde.binary,
        // Set the output serde here
        output: restate.serde.binary,
      },
      async (ctx: Context, data: Uint8Array): Promise<Uint8Array> => {
        // Process the request
        return data;
      }
    ),
  },
});
```

---

## Error Handling

Throw `TerminalError` to stop retries and propagate failure permanently:

```ts
throw new TerminalError("Something went wrong.", { errorCode: 500 });
```


All other exceptions are retried with exponential backoff by default, and eventually paused.

Catch `TerminalError` from `ctx.run` to handle permanent failures and execute compensations (see sagas).

---

## SDK Clients (External Invocations)

Use `@restatedev/restate-sdk-clients` to call Restate handlers from outside a Restate context (e.g., from a REST API, a script, or a cron job):

```ts
const restateClient = clients.connect({ url: "http://localhost:8080" });

// Request-response
const result = await restateClient
  .serviceClient<MyService>({ name: "MyService" })
  .myHandler("Hi");

// One-way
await restateClient
  .serviceSendClient<MyService>({ name: "MyService" })
  .myHandler("Hi");

// Delayed
await restateClient
  .serviceSendClient<MyService>({ name: "MyService" })
  .myHandler("Hi", clients.rpc.sendOpts({ delay: { seconds: 1 } }));
```

---

## TypeScript-Specific Pitfalls

- **Use `RestatePromise`**, NOT native `Promise`. Native promises break deterministic replay.
- **Return values from `ctx.run()` must be JSON-serializable** (no functions, circular references, or class instances without custom serde).
- **Never use `setTimeout`, `Math.random()`, `Date.now()`, or `new Date()`** -- use Restate Context actions instead.
- **Never use global mutable variables for state** -- use Restate's K/V store for durable state.
- **For detailed API reference:** use the MCP server or TSDocs.

## Testing

Install: `npm install --save-dev @restatedev/restate-sdk-testcontainers`

Tests run against a real Restate Server in Docker via Testcontainers. 

```typescript
import {RestateTestEnvironment, TestEnvironmentOptions} from "@restatedev/restate-sdk-testcontainers";
import * as clients from "@restatedev/restate-sdk-clients";
import { describe, it, beforeAll, afterAll, expect } from "vitest";
import { greeter } from "./greeter-service";

describe("MyService", () => {
  let restateTestEnvironment: RestateTestEnvironment;
  let restateIngress: clients.Ingress;

  beforeAll(async () => {
    restateTestEnvironment = await RestateTestEnvironment.start({
      services: [greeter],
      alwaysReplay: true,
    } as TestEnvironmentOptions);
    restateIngress = clients.connect({ url: restateTestEnvironment.baseUrl() });
  }, 20_000);

  afterAll(async () => {
    await restateTestEnvironment?.stop();
  });

  it("Can call methods", async () => {
    const client = restateIngress.objectClient(greeter, "myKey");
    await client.greet("Test!");
  });

  it("Can read/write state", async () => {
    const state = restateTestEnvironment.stateOf(greeter, "myKey");
    await state.set("count", 123);
    expect(await state.get("count")).toBe(123);
  });
});
```

Use tests also to catch non-determinism bugs that unit tests miss: if handler code is non-deterministic, replay produces different results and the test fails.

---

## Further resources

- For detailed API: use the TSDocs https://restatedev.github.io/sdk-typescript/ or the bundled restate-docs MCP server
- Examples: https://github.com/restatedev/examples
- AI agent examples: https://github.com/restatedev/ai-examples