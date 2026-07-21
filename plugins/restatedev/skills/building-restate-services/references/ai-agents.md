# Building AI Agents with Restate

Load this reference for every AI agent task, then load the framework-specific reference when one exists.

## Choose the durability boundary

Prefer the narrowest integration that journals each expensive or side-effecting step:

| Integration | Use it when | Tradeoff |
| --- | --- | --- |
| Restate framework middleware | The framework has an official Restate integration | Journals model calls and preserves framework ergonomics. Tools still need explicit durable steps. |
| Explicit agent loop | You control model and tool dispatch | Wrap every model call and tool side effect in `ctx.run()` or `ctx.run_typed()`. This gives the clearest recovery and observability. |
| Entire agent in one context action | No fine-grained integration is possible | Coarse recovery only. A failure reruns the whole agent action, and no Restate context operation may run inside it. |

Do not call an unsupported agent framework directly from replayable handler code. Either integrate its model and tool boundaries or treat the entire execution as one context action.

## Choose the Restate service type

- Use a **Service** for independent, single-turn agent requests.
- Use a **Virtual Object** keyed by conversation or session ID for persistent chat history and single-writer session coordination.
- Use a **Workflow** keyed by goal or job ID for a run with a defined start, finish, and interaction handlers.
- Put independently deployed agents and complex tools behind separate Restate services. Invoke them with typed clients so calls, retries, and queues remain durable.
- Use an awakeable for human approval or an external callback. The handler suspends without holding compute and resumes when the awakeable resolves.

## Make every boundary durable

- Journal every model response. Replaying a handler must reuse the response instead of paying for and potentially receiving a different completion.
- Wrap network, database, file, vector-store, and other side effects inside tools with a context action.
- Persist built-in framework tool results such as web search. Do not assume the framework makes them replay-safe.
- Never call Restate context operations from inside `ctx.run()` or `ctx.run_typed()`.
- Use deterministic Restate helpers for time, UUIDs, randomness, sleeps, and concurrency.

## Control retries and cost

- Set a bounded retry policy for model calls and expensive tools. Do not inherit infinite retries for paid APIs without an explicit decision.
- Convert invalid input, denied actions, unsupported requests, and other permanent failures to terminal errors.
- Use idempotency keys at ingress when a caller may resubmit the same agent job.
- Use Restate 1.7 scopes and limit keys for tenant, team, or user concurrency budgets. Load `references/flow-control-and-scopes.md` before choosing those identities.
- Add a maximum agent-step count or another termination condition so a model cannot loop indefinitely.

## Manage sessions deliberately

- Keep session state in a Virtual Object or an external store accessed through durable steps, not process memory.
- Store typed messages and preserve tool-call metadata required by the framework.
- Bound history growth with summarization, compaction, or archival. Journal the summarization call like any other model call.
- Keep large documents and embeddings outside Restate K/V state. Store durable references to them when appropriate.

## Preserve replay determinism

- Use Restate concurrency primitives instead of `Promise.all`, `asyncio.gather`, goroutines, or framework-native parallel tool execution unless the official Restate middleware explicitly linearizes it.
- Propagate Restate suspension and terminal errors. Do not turn them into tool-result text for the model.
- Treat framework-generated IDs, timestamps, and routing choices as non-deterministic unless they are journaled.
- Do not stream a model response from inside a context action. Journal the completed model result, then use a documented session or pub/sub pattern when clients need resumable progress.

## Test and operate

- Add an always-replay Testcontainers test for every handler-logic change.
- Test concurrent calls to the same session and separate session keys.
- Test retry exhaustion, terminal tool failures, cancellation, and human-approval timeouts.

Implementation guides:

- Durable agents: https://docs.restate.dev/ai/patterns/durable-agents
- Sessions: https://docs.restate.dev/ai/patterns/sessions
- Tools and sub-workflows: https://docs.restate.dev/ai/patterns/tools
- Error handling: https://docs.restate.dev/ai/patterns/error-handling
- Human in the loop: https://docs.restate.dev/ai/patterns/human-in-the-loop
- Agent framework integration requirements: https://docs.restate.dev/ai/sdk-integrations/integration-guide
