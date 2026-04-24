# Tour of Microservice Orchestration

A guided walkthrough of several services coordinated durably via Restate. Canonical docs: `https://docs.restate.dev/tour/microservice-orchestration`.

## Starter template

Run in an empty directory:

| Language | Command |
|---|---|
| TypeScript | `restate example typescript-tour-of-orchestration` |
| Python | `restate example python-tour-of-orchestration` |
| Java | `restate example java-tour-of-orchestration` |
| Go | `restate example go-tour-of-orchestration` |

`cd` in, install deps, start the service. Register: `restate deployments register http://localhost:9080`.

The starter typically ships with an order / booking / payment style scenario spanning multiple services and virtual objects. Every stage below works against that same codebase.

## How to run each stage

1. Find the relevant file in the starter (`README.md` or `ls -R`).
2. Read it — cite lines, don't paste.
3. Run the curl/CLI command — one at a time.
4. Describe what to observe.
5. Offer a small modification ("try it yourself").

## Stage 1 — Basic orchestration

**Concept:** A service calls another service via `ctx.serviceClient(...)`. The call goes through the Restate server, so it's journaled on both sides. If the caller crashes mid-call, it replays and the response is retrieved from the journal.

**Run:** invoke the top-level service with curl. Watch the logs of the downstream services.

**Try:** add a `console.log`/`print` inside a downstream handler and observe it fires once across caller retries.

## Stage 2 — Durable execution across services

**Concept:** The guarantee from stage 1 doesn't depend on both services being healthy at the same time. If the downstream service is down, the caller's `ctx.serviceClient(...)` call retries transparently; once the downstream comes back, the call completes.

**Run:** kill the downstream service. Invoke the top-level service. Wait. Restart the downstream service.

**Observe:** the top-level call completes — no manual retry.

**Try:** kill the **top-level** service instead of the downstream one during a call. Restart it. The journal picks up where it left off.

## Stage 3 — Error handling

**Concept:** Transient errors (regular throws) are retried per the service's retry policy. `TerminalError` stops retries and fails the call up to the caller, which also sees it as a terminal error.

**Run:** force the downstream service to throw a transient error N times, then succeed. Force it to throw a `TerminalError`. Compare journal behavior in the admin UI.

**Try:** add a retry policy override for one handler (if the SDK supports it in the starter); verify retries stop earlier.

## Stage 4 — Sagas across services

**Concept:** When a multi-step orchestration fails partway, compensations run in reverse. Each compensation is its own service call — also journaled, also retried durably. Partial rollback is safe to crash through.

**Run:** force a failure after the second step. Watch the compensation for step 1 run, then the overall call fail.

**Try:** add a third forward step with a compensation. Force a failure after it.

## Stage 5 — Virtual Objects

**Concept:** A Virtual Object is a stateful entity keyed by id. Restate ensures **at most one handler per key runs at a time**, so you get serialized access without manual locking. Each key has its own state map.

**File to read:** the starter's account/cart/room virtual object.

**Run:** hit the same key concurrently (fire two curl calls in parallel). Watch them serialize. Inspect state via `restate kv get <service> <key>`.

**Try:** hit two **different** keys in parallel — they run in parallel.

## Stage 6 — Resilient communication

**Concept:** Three patterns, all journaled:

- **Request-response**: `ctx.serviceClient(...).handler(...)` — await the result.
- **One-way send**: `ctx.serviceSendClient(...).handler(...)` — fire and forget, durable.
- **Delayed send**: `ctx.serviceSendClient(...).handler(...).withDelay(...)` — fires at least N later, survives restarts.

**Run:** swap one call between all three patterns; watch the journal differences.

**Try:** schedule a delayed send, then restart both services. The send still fires at the right time.

## Stage 7 — Request idempotency

**Concept:** Attach an idempotency key to a request (header or CLI flag). Duplicate requests with the same key return the original result instead of running again. Survives client retries.

**Run:**

```bash
curl -s -H 'idempotency-key: order-42' \
  localhost:8080/OrderService/submit -d '{"items":[...]}'
```

Run the same curl twice. Second call returns the first call's result.

**Try:** change the body but reuse the key — the first result still comes back (idempotency is keyed by key, not body).

## Stage 8 — External events (awakeables)

**Concept:** A handler creates an awakeable and blocks on it. The awakeable's id is an external handle — some other system resolves it later via an HTTP call. The handler unblocks with that result.

**Run:** invoke a handler that creates an awakeable. Grab the awakeable id from logs or the UI. Resolve it:

```bash
curl -s localhost:8080/restate/awakeables/<awakeable-id>/resolve \
  -H 'Content-Type: application/json' -d '"payload"'
```

**Observe:** the original handler unblocks.

**Try:** reject an awakeable with `/reject` and have the handler surface that as a `TerminalError`.

## Stage 9 — Durable timers

**Concept:** `ctx.sleep(duration)` suspends the handler; the server wakes it at the right time. Works across services, across restarts.

**Run:** hit a handler that sleeps for 30s. Kill the service. Restart after 10s. The sleep still completes at the original wake time.

**Try:** race `ctx.sleep` against an awakeable to build a "wait up to 5 minutes for confirmation" pattern.

## Stage 10 — Concurrent tasks

**Concept:** Fan-out to N calls in parallel, then join. Use the SDK-provided combinator (`RestatePromise.all` in TS, similar in other SDKs) — **not** language-native `Promise.all` / `asyncio.gather`. The SDK combinator is deterministic across replay.

**Run:** hit a handler that fans out to 3 downstream services and waits for all three.

**Observe:** the three downstream invocations run in parallel; the parent blocks until all complete.

**Try:** switch to a "first to complete wins" combinator (`RestatePromise.race` or equivalent). Observe behavior.

## Handoff

After stage 10:

- *"Want to build your own orchestration?"* → hand off to the sibling `building-restate-services` skill.
- *"Want to see a single long-running workflow instead?"* → offer the Tour of Workflows (`tour-of-workflows.md`).
- Deployment, Kafka, server config → `restate-docs` MCP server.
- More examples → `github.com/restatedev/examples`.
