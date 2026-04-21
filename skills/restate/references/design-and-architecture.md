# Design and Architecture

## Service type decision framework

Choose the service type based on state needs, concurrency requirements, and the nature of the entity.

| Type | State | Concurrency | When to use |
|------|-------|-------------|-------------|
| **Service** | None | Unlimited parallel | Stateless handlers: API endpoints, ETL pipelines, sagas, background jobs |
| **Virtual Object** | K/V per key | Single writer per key + concurrent readers | Stateful entities: user sessions, shopping carts, device twins, counters |
| **Workflow** | Per workflow ID | One `run` + concurrent interaction handlers | Linear processes with a start and end: approvals, onboarding, order fulfillment |

Concrete examples:
- A payment processor that calls Stripe and updates a database is a **Service** (stateless, each call independent).
- A shopping cart that tracks items per user is a **Virtual Object** keyed by user ID (persistent state, single-writer mutations).
- An order approval flow that waits for manager sign-off is a **Workflow** (exactly-once per order, pauses for external signal).

### Handler types

**Virtual Object handlers:**
- **Exclusive** (default): Processes one invocation at a time per key. Use for state mutations.
- **Shared**: Runs concurrently across all invocations for a key. Use for read-only queries.

**Workflow handlers:**
- **`run`**: The main handler. Executes exactly once per workflow ID. Equivalent to exclusive.
- **Interaction handlers**: Shared handlers that run concurrently. Use to send signals or query status while `run` is active.

### Workflow vs Virtual Object

- Use a **Workflow** for linear processes with a clear start and end (order processing, user onboarding, approval chains). The `run` handler executes exactly once per ID and can pause for external signals via durable promises.
- Use a **Virtual Object** for long-lived entities with no fixed lifecycle (user profiles, device state, counters). The object persists indefinitely and handles an unbounded stream of requests.
- If the process restarts periodically or needs multiple independent runs, use a Virtual Object with a `run` method pattern rather than a Workflow.

## Virtual Object keying strategy

The key is the entity identity. All state for a key is isolated from other keys. Exclusive handlers queue per key.

### Choosing good keys

Pick keys that distribute load evenly across instances:
- **E-commerce**: order ID, user ID, product ID
- **Messaging**: session ID, conversation ID, channel ID
- **IoT**: device ID, sensor ID
- **Finance**: account ID, transaction ID
- **Gaming**: player ID, match ID

### Key cardinality rules

One key per logical entity that needs its own state and concurrency boundary.

### Anti-patterns

- Keying on low-cardinality values like "region" or "environment" (3 keys = 3 concurrent exclusive handlers total).
- Keying on high-cardinality ephemeral values like request ID when state should be grouped by user.
- Using composite keys (e.g., `user:order:item`) when a simpler key with structured state suffices.

## Exclusive vs shared handler design

### Exclusive handlers (default)

- Single writer per key. Invocations queue and execute one at a time.
- Use for: state mutations (`ctx.set`), state transitions, operations requiring consistency.
- Keep exclusive handlers short. Long-running exclusive handlers block all other exclusive calls for that key.

### Shared handlers

- Run concurrently with no queuing.
- Use for: read-only queries (`ctx.get`), status checks, health probes.
- Must not mutate state. Calling `ctx.set` from a shared handler is a programming error.

### Workflow handler mapping

- The `run` handler is exclusive (executes once per workflow ID).
- Interaction handlers are shared (concurrent). Use them to resolve durable promises or read workflow state while `run` is active.

### Design rules

- If a handler calls `ctx.set()` -> make it exclusive.
- If a handler only calls `ctx.get()` -> make it shared.
- Minimize work in exclusive handlers. Move read-only logic to shared handlers to reduce queue wait times.
- Split large exclusive handlers: perform the state mutation in a short exclusive handler, then delegate long-running work to a Service call or a one-way send.

## Deadlock prevention

Deadlocks occur when exclusive handlers on Virtual Objects form circular call chains. Restate cannot detect or break these automatically.

### Deadlock types

**Cross-deadlock**: Object A (key1) exclusive handler calls Object B (key2) exclusive handler, while Object B (key2) exclusive handler calls Object A (key1). Both block waiting for each other.

**Cycle deadlock**: A -> B -> C -> A, where each arrow is an exclusive handler call. Any cycle of exclusive calls across Virtual Objects deadlocks.

**Self-deadlock**: An exclusive handler on Object A (key1) calls another exclusive handler on the same Object A (key1). The second call queues behind the first, which is waiting for the second to complete.

### Prevention strategies

1. **Map the call graph before implementing.** Draw which handlers call which. If any cycle exists among exclusive handlers, redesign.
2. **Use one-way sends to break cycles.** A one-way send (`ctx.send()`) does not block. The caller continues immediately.
3. **Use shared handlers for reads.** If the call only needs to read state, make the target handler shared. Shared handlers do not queue.
4. **Use a Service as intermediary.** Services have no keyed queues. Place coordination logic in a Service that calls Virtual Objects, avoiding direct Object-to-Object exclusive cycles.

## Communication patterns

Restate provides several durable communication mechanisms. All survive crashes and process restarts.

| Pattern | Mechanism | Behavior |
|---------|-----------|----------|
| **Request-response** | Durable RPC (service call) | Caller blocks until callee returns. Retried automatically on failure. |
| **One-way message** | `ctx.send()` | Fire-and-forget. Caller continues immediately. Message is delivered durably. |
| **Delayed message** | `ctx.send()` with delay | Message delivered after a specified duration. Durable timer, survives crashes. |
| **Awakeable** | `ctx.awakeable()` + external `resolve`/`reject` | Handler pauses, external system signals completion via HTTP. Use for human-in-the-loop, webhooks, third-party callbacks. |
| **Durable promise** | Workflow promise `resolve`/`reject` | Cross-handler signaling within a Workflow. `run` handler awaits, interaction handler resolves. |

### When to use which

- **Request-response**: Default for service-to-service calls where the caller needs the result.
- **One-way message**: When the caller does not need the result or wants to avoid blocking (break deadlock cycles, trigger background work).
- **Delayed message**: Scheduled future execution (reminders, timeouts, retry after delay). Combine with self-invocation for cron patterns.
- **Awakeable**: Integration with external systems that call back via HTTP (payment processors, human approvals, third-party webhooks).
- **Durable promise**: Coordination between a Workflow's `run` handler and its interaction handlers.

Common mistakes:
- Do NOT use a durable promise to cancel an invocation. Use the Admin cancel API (`ctx.cancel(id)`) instead. See `references/lifecycle-and-operations.md`.
- Do NOT poll a state flag to wait for a result. Use `ctx.attach(invocationId)` to wait for a workflow result.
- Do NOT use `ctx.sleep()` + send to schedule future work. Use a delayed send directly (it does not block the handler).

For detailed SDK-specific API signatures: use the bundled restate-docs MCP server. For cancellation, idempotency, attach, and retention: see `references/invocation-lifecycle.md`.

## Common architecture patterns

### Saga (compensating transactions)

Execute a sequence of steps. Track completed steps. On `TerminalError`, execute compensating actions in reverse order. Each step and compensation is a `ctx.run()` call.

### Fan-out / fan-in

Launch parallel work using Restate concurrency combinators (`CombineablePromise.all` in TypeScript, `restate.gather` in Python, `DurableFuture.all` in Java, `restate.Wait` in Go). Each parallel branch is a durable service call. Collect results after all branches complete.

### Cron (self-scheduling)

A handler performs work, then sends a delayed message to itself. On the next invocation, it repeats. This creates a durable cron loop with no external scheduler. Use a Virtual Object to store cron state (last run, interval).

### Event enrichment

A Virtual Object receives events (via Kafka ingestion or one-way sends), accumulates state from each event, and produces enriched output. The single-writer guarantee on exclusive handlers ensures consistent state without locks.

### State machine

A Virtual Object where exclusive handlers represent transitions. State is stored via `ctx.set()`. Each handler validates the current state, performs the transition, and updates state. Shared handlers expose current state for queries.

For runnable examples and templates, see:
- General patterns: https://github.com/restatedev/examples
- AI agent examples: https://github.com/restatedev/ai-examples

## AI agent architecture

### Stateless agents

Use a **Service**. Each invocation is independent. Suitable for single-turn request-response agents with no conversation history.

### Stateful agents (sessions)

Use a **Virtual Object** keyed by session ID. Store conversation history in K/V state. The exclusive handler guarantee ensures only one LLM call per session at a time, preventing race conditions in conversation state.

### Framework selection

- **TypeScript**: Vercel AI SDK with `@restatedev/restate-ai` (load `references/ts/restate-vercel-ai-agents.md`)
- **Python**: OpenAI Agents SDK, Google ADK, or Pydantic AI with Restate wrappers (load the corresponding `references/python/` file)
- **Any language**: Raw Restate SDK with manual LLM API calls wrapped in `ctx.run()`

### Durability requirement

Every LLM call and tool execution must be durable (wrapped in `ctx.run()` or handled via framework middleware). Without durability, a crash mid-conversation loses the LLM response and any tool side effects.

For runnable AI agent examples: https://github.com/restatedev/ai-examples
