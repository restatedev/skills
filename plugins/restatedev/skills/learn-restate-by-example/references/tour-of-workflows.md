# Tour of Workflows

A guided walkthrough of a signup workflow that evolves through 10 stages. Canonical docs: `https://docs.restate.dev/tour/workflows`.

## Starter template

Run in an empty directory:

| Language | Command |
|---|---|
| TypeScript | `restate example typescript-tour-of-workflows` |
| Python | `restate example python-tour-of-workflows` |
| Java | `restate example java-tour-of-workflows` |
| Go | `restate example go-tour-of-workflows` |

`cd` in, install deps, start the service. Register once: `restate deployments register http://localhost:9080`.

The starter contains the workflow plus stub services (email, user DB). Each stage below builds on the previous one — keep the same codebase open.

## How to run each stage

At the top of each stage:

1. **Find the relevant file** in the starter. The file layout differs by language; use the `README.md` or `ls -R` to orient.
2. **Read the code** — cite lines, don't paste.
3. **Run the stage's curl/CLI command** — one at a time.
4. **Describe what to observe** — output, logs, admin UI journal entries.
5. **Offer a small modification** — the "try it yourself" line.

## Stage 1 — Basic workflow

**Concept:** A workflow is a handler decorated/annotated as a workflow. Each workflow run is identified by a *workflow id*. Re-invoking with the same id attaches to the existing run (and can fetch its result), not start a new one.

**Run:**

```bash
curl -s localhost:8080/SignupWorkflow/alice/run \
  -H 'Content-Type: application/json' \
  -d '{"email":"alice@example.com"}'
```

**Observe:** returns the final result, or attaches if already running. The admin UI shows one invocation keyed by `alice`.

**Try:** invoke with the same id twice → second call attaches to the first.

## Stage 2 — Durable execution

**Concept:** Steps wrapped in `ctx.run(...)` are journaled. After a crash, the handler replays; journaled steps return their recorded result without re-executing.

**Run:** kill the service mid-invocation (Ctrl+C) → restart it → the workflow resumes on the next handler tick.

**Observe:** side-effect logs inside `ctx.run` appear **once**, not on every replay.

**Try:** add a `ctx.run("log", () => console.log("only once"))` step and verify the log only appears the first time.

## Stage 3 — Inline steps vs. activity services

**Concept:** Short, in-process steps → `ctx.run`. Heavy, reusable work (shared with other workflows, different scaling, different failure domain) → a separate Restate service invoked with `ctx.serviceClient(...)`.

**File to read:** whichever activity the starter extracts (often `EmailService` or similar).

**Run:** invoke the workflow; watch both services get journal entries.

**Try:** convert one inline `ctx.run` into an activity call, and one activity call into an inline step. Note the tradeoffs: determinism, reuse, operational boundary.

## Stage 4 — Workflow state

**Concept:** Inside a workflow, `ctx.set` / `ctx.get` persist key-value state tied to the workflow id. That state is queryable from outside.

**Run (from another terminal, workflow still running):**

```bash
curl -s localhost:8080/SignupWorkflow/alice/getStatus
```

Or via the CLI state inspector: `restate kv get SignupWorkflow alice`.

**Observe:** whatever status the workflow most recently `set`.

**Try:** add a new state field (e.g. `lastEmailSentAt`) and query it.

## Stage 5 — Signals (awaiting external input)

**Concept:** A workflow can block on a signal using `ctx.promise(...)` / `ctx.awakeable(...)`. External code resolves it via an HTTP call, which unblocks the workflow.

**Run:** invoke the workflow. It blocks. Then:

```bash
curl -s localhost:8080/SignupWorkflow/alice/confirmEmail \
  -H 'Content-Type: application/json' \
  -d '{"token":"xyz"}'
```

**Observe:** the workflow unblocks and continues.

**Try:** race the signal against a timer (see stage 7) — whichever resolves first wins.

## Stage 6 — Emitting events

**Concept:** A workflow can send one-way messages (`ctx.serviceSendClient(...)`) or publish to Kafka (via a Kafka-backed service). Sends are journaled, so they survive crashes and never fire twice.

**Run:** invoke the workflow; check the downstream service's logs.

**Try:** change a fire-and-forget call to a request-response call (`serviceClient` instead of `serviceSendClient`). Note when you want each.

## Stage 7 — Durable timers

**Concept:** `ctx.sleep(duration)` is a journaled timer. The handler suspends; the server wakes it at the right time. The process can be killed and restarted during the sleep — the timer persists.

**Run:** invoke a path that sleeps. Kill the service. Restart. Watch the timer fire anyway.

**Try:** combine `ctx.sleep` with a signal (`Promise.race` / equivalent) to build a "wait up to N minutes for email confirmation".

## Stage 8 — Errors: transient vs. terminal

**Concept:** Plain thrown errors are **transient** — Restate retries forever (or per the retry policy). Throwing a `TerminalError` stops retries and fails the invocation permanently. Inside `ctx.run`, both behaviors are respected.

**Run:** inject a transient failure (throw once, then succeed). Inject a terminal failure (`throw new TerminalError(...)`). Observe retry vs. no-retry behavior in the journal.

**Try:** wrap a call in try/catch, translate a 4xx HTTP response into a `TerminalError` and a 5xx into a transient one.

## Stage 9 — Sagas

**Concept:** Compensations are just `ctx.run` steps you register as you go. On failure, run the registered compensations in reverse order. The starter has a helper for this pattern.

**Run:** force a later step to fail; watch the compensations execute. Each compensation is itself journaled — safe to crash mid-rollback.

**Try:** add a new forward step and its compensation. Force a failure after it. Verify both compensations run.

## Stage 10 — Cancellation

**Concept:** `restate invocations cancel <id>` (or the admin UI button) cancels a running workflow. In-flight `ctx.run` finishes; pending sleeps/signals abort; compensations run.

**Run:**

```bash
restate invocations list
restate invocations cancel <invocation-id>
```

**Observe:** the workflow terminates; compensations from stage 9 run.

**Try:** start a long-sleeping workflow, cancel it, confirm the sleep is aborted rather than waited out.

## Bonus — Deploying to serverless

**Concept:** A Restate service endpoint is just a handler that speaks the Restate protocol. It can run in Lambda, Cloudflare Workers, Fly, etc. — the server sends it requests; the handler responds.

Don't run this stage locally. Instead:

- Read the deploy section in the starter's `README.md`.
- Point at the `restate-docs` MCP server for platform-specific deploy guides.

## Handoff

After stage 10:

- *"Ready to build your own workflow?"* → hand off to the sibling `building-restate-services` skill.
- *"Want to see how this looks across multiple services?"* → offer the Tour of Microservice Orchestration (`tour-of-orchestration.md`).
- Point at `github.com/restatedev/examples` for more workflow patterns.
