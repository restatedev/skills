# TypeScript Durable Coroutines

## When to use it

Use `@restatedev/restate-sdk-gen` for concurrent routines that each contain multiple durable steps within one invocation. Examples include parallel agent tools with guardrails, subagents sharing a sandbox connection, or background work coordinated with user input.

Use the standard SDK for sequential logic, parallel individual steps, or concurrent calls to separate Restate handlers. Coroutines work in Services, Virtual Objects, and Workflows without changing state access or handler exclusivity rules.

## Setup

Start with the template:

```bash
npx @restatedev/restate example typescript-hello-world-gen
```

For an existing project:

```bash
npm install @restatedev/restate-sdk @restatedev/restate-sdk-gen
```

Use the coroutine package namespace for generator operations. Import the `Operation` type from that package and `TerminalError` from `@restatedev/restate-sdk` when needed.

```ts
import * as restate from "@restatedev/restate-sdk-gen";
```

## Use in existing handlers

Call `execute(ctx, operation)` from an existing async handler. It returns a promise with the result. Use this to add coroutines without rewriting the rest of the handler.

```ts
import * as sdk from "@restatedev/restate-sdk";
import * as restate from "@restatedev/restate-sdk-gen";

export const existingService = sdk.service({
  name: "ExistingService",
  handlers: {
    summary: async (ctx: sdk.Context, userId: string) =>
      restate.execute(ctx, loadUserSummary(userId)),
  },
});
```

Use `yield*` inside a coroutine to retrieve an operation's result. `fetchUser` and `fetchOrders` below stand in for application APIs.

Create a reusable operation with a generator factory:

```ts
type UserSummary = { name: string; orderCount: number };

function loadUserSummary(userId: string): Operation<UserSummary> {
  return restate.gen(function* () {
    const user = yield* restate.run(() => fetchUser(userId), {
      name: "fetch-user",
    });
    const orders = yield* restate.run(() => fetchOrders(user.id), {
      name: "fetch-orders",
    });
    return { name: user.name, orderCount: orders.length };
  });
}
```

- Keep external I/O and nondeterminism inside `run`. Its callback can use native async/await.
- Give every `run` a deterministic journal name using `{ name: "..." }`, or pass a named function. Anonymous callbacks require an explicit name.
- Do not call coroutine operations or context methods from a `run` callback.
- Pass the callback's `{ signal }` to APIs that accept an `AbortSignal` for cancellation.
- Configure bounded retries for expensive operations through `run` options: `retry: { maxAttempts, maxDuration, initialInterval, maxInterval, exponentiationFactor }`. Exhaustion throws `TerminalError` at the yield site. Throw `TerminalError` for permanent failures.

## Concurrent execution

`Operation<T>` is lazy. `gen` creates a fresh generator each time the operation executes, so executing it again repeats its body. A raw generator instance can only be consumed once. `Future<T>` remembers its outcome and can be yielded repeatedly; `run` and `sleep` start eagerly. `spawn` starts an operation and returns a `Task<T>`, a future with an `interrupt` method.

Inside an existing handler, spawn one routine per input and join the results:

```ts
summaries: async (ctx: sdk.Context, userIds: string[]) =>
  restate.execute(ctx, restate.gen(function* () {
    const tasks = userIds.map((id) => restate.spawn(loadUserSummary(id)));
    return yield* restate.all(tasks);
  })),
```

Use coroutine combinators, not native `Promise` combinators or standard SDK `RestatePromise`, for these futures:

| Combinator | Behavior |
| --- | --- |
| `all` | Results in input order; throws if an input fails. |
| `race` | First result or error. |
| `any` | First successful result; `AggregateError` if all fail. |
| `allSettled` | Every result or error in input order. |
| `select` | The first branch's `tag` and `future`; yield the future to retrieve its outcome. |

For timeouts, combine the work with a durable timer and inspect the tag:

```ts
const summary = restate.spawn(loadUserSummary(userId));
const winner = yield* restate.select({
  done: summary,
  timeout: restate.sleep({ seconds: budgetSeconds }),
});
if (winner.tag === "timeout") {
  throw new TerminalError("Loading the summary timed out");
}
return yield* winner.future;
```

A race does not interrupt losing tasks or undo external work. Only one coroutine executes JavaScript at a time; I/O can overlap. Generators let the SDK control resumption for deterministic replay.

## Lifetime and cancellation

By default, the main operation returning or throwing abandons unfinished coroutines. Abandoned coroutines do not run `catch` or `finally`. Yield tasks before returning when their results or cleanup matter. Use a durable service send for work that must outlive this invocation.

`task.interrupt()` injects `InterruptedError` at the next yield, aborts the task's in-flight `run` signal, and propagates to its spawned descendants. Pass that signal to external APIs that support cancellation; I/O that ignores it can continue. A task can catch the interruption and recover. Interrupt, then yield the task to let cleanup finish:

```ts
const work = restate.gen(function* () {
  try {
    yield* restate.sleep({ minutes: 1 });
  } finally {
    yield* restate.run(() => recordAudit("cleanup", userId), {
      name: "cleanup",
    });
  }
});
const task = restate.spawn(work);

task.interrupt();
try {
  yield* task; // Wait for the finally block to finish.
} catch (error) {
  if (!(error instanceof restate.InterruptedError)) throw error;
}
```

For cooperative cancellation, create a `channel<void>()`, pass it to the worker, and include `stop.receive` in `select`. The sender must `yield* stop.send()` and then yield the worker. The worker chooses where to stop and can return partial results. This does not abort its current external call.

Channels resolve once, retain the same resolved receive future, and ignore later sends. They are local to an invocation and reconstructed during replay. For external stop requests, receive a signal or awakeable first, then send on the local channel from generator code. A channel is not an external event API.

## Context local values

Define a `contextLocal<T>()` handle at module scope and use `get`/`set` in generator code. It shares values across all coroutines in one `execute` call while isolating separate calls. Each generator handler invocation uses its own `execute` call. A default is optional; without one, an unset value is `undefined`.

```ts
const requestId = restate.contextLocal<string>("unknown");

function loadAuditedSummary(userId: string): Operation<UserSummary> {
  return restate.gen(function* () {
    const id = requestId.get();
    yield* restate.run(() => recordAudit(id, userId), { name: "audit" });
    return yield* loadUserSummary(userId);
  });
}
```

Set it before spawning routines:

```ts
requestId.set(id);
const tasks = userIds.map((userId) =>
  restate.spawn(loadAuditedSummary(userId))
);
return yield* restate.all(tasks);
```

Read the value before entering a `run` callback; `get` and `set` cannot run inside that callback. Set values deterministically from input or journaled results so replay reconstructs them. All coroutines share writes to the same handle; this is not per-coroutine storage. Use `state` for data that must persist across invocations. Recreate local connections and caches after restart.

## Generator handlers and other APIs

You can also define whole handlers as generators with the coroutine package's `service`, `object`, and `workflow` helpers. Generator handlers receive input directly, without `ctx`. Serve these definitions with `serve` from `@restatedev/restate-sdk`.

```ts
export const userService = restate.service({
  name: "UserService",
  handlers: {
    *greet(userId: string): Operation<string> {
      const user = yield* restate.run(() => fetchUser(userId), {
        name: "fetch-user",
      });
      return `Hello, ${user.name}!`;
    },
  },
});
```

The optional third argument to `execute`, `{ onMainExit: "join" }`, waits for all spawned coroutines; a task that never finishes then keeps the invocation running.

Use the coroutine package's `client`/`sendClient` for calls from coroutines and `clients.connect` with `clients.client`/`clients.sendClient` for external calls. Do not assume standard SDK ingress clients infer generator-handler types. Other primitives include `state`, `sharedState`, `sleep`, `awakeable`, `signal`, and `workflowPromise`. Use exact signatures from the installed package or current docs; older guide examples may use obsolete names.

## Validation and reference

Typecheck snippets against the installed coroutine package. Test handler changes with `RestateTestEnvironment.start({ services, alwaysReplay: true })`. Cover concurrency, terminal failures, timeouts, cleanup after interruption, and context isolation where relevant.

- Documentation: https://docs.restate.dev/develop/ts/concurrent-tasks#durable-coroutines
- API and tutorial: https://github.com/restatedev/sdk-typescript/tree/main/packages/libs/restate-sdk-gen
