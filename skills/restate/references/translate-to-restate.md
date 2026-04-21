# Translate to Restate

Use this reference when converting projects from other workflow orchestrators or application architectures to Restate.

## Analysis checklist

Before migrating, identify the following in the existing codebase:

- [ ] **API handlers/endpoints**: List all HTTP routes, gRPC services, or message consumers. Note which are stateless vs stateful.
- [ ] **Stateful entities**: Identify users, orders, sessions, carts, devices, or any entity with mutable state. Note where state lives (database, cache, in-memory).
- [ ] **Communication patterns**: Map how components talk to each other (HTTP calls, message queues, event buses, direct function calls). Note which are synchronous vs asynchronous.
- [ ] **Background jobs**: List cron tasks, scheduled work, async job queues, and delayed processing. Note scheduling mechanisms (crontab, CloudWatch Events, Celery beat, Bull queues).
- [ ] **External calls needing durability**: Identify API calls, database writes, payment processing, or third-party integrations that must not be duplicated on retry.
- [ ] **Multi-step processes**: Find workflows, sagas, state machines, or approval flows with failure handling or compensation logic.
- [ ] **Failure handling**: Catalog existing retry logic, circuit breakers, dead letter queues, and manual compensation code. These are prime candidates for replacement with Restate's built-in mechanisms.
- [ ] **State consistency requirements**: Identify where race conditions, double-processing, or lost updates are current problems or risks.

## Workflow orchestrator concept mapping

Map concepts from existing orchestrators (Temporal, Step Functions, etc.) to Restate equivalents.

| Orchestrator concept | Restate equivalent                                                                                                 | Do NOT reinvent |
|---|--------------------------------------------------------------------------------------------------------------------|---|
| Activity / Task / Step | `ctx.run()`                                                                                                        | Manual retry loops, external task queues |
| Workflow / Process / State Machine | Workflow `run` handler or Service handler                                                                          | JSON/YAML workflow definitions, state machine libraries |
| Signal (data) / Message Event | Durable Promise or Awakeable                                                                                       | Polling a state flag in a loop |
| Signal (cancel) / Stop workflow | Admin cancel API: `ctx.cancel(id)`                                                                                 | Cooperative cancel flag + poll pattern |
| Query / Read state | Shared handler on Virtual Object / Durable Promise or state get on Workflow                                        | Polling handler, state-dump endpoint |
| Timer / Sleep | `ctx.sleep()` or delayed send                                                                                      | `setTimeout`, native sleep, external scheduler |
| Child Workflow / Sub-process | Service-to-service call (durable RPC)                                                                              | HTTP calls with manual retry logic |
| Workflow state / Variables | Virtual Object K/V or Workflow state                                                                               | External database for workflow-scoped data |
| Worker / Task Queue | Restate Server + service endpoint                                                                                  | Worker pools, polling loops, task queue libraries |
| Retry policy | Service/handler retry policy or `ctx.run()` retry options. (see https://docs.restate.dev/guides/error-handling.md) | Retry counter in state, try-catch retry loop |
| Saga / Compensation | Try/catch with compensation list                                                                                   | Separate compensation service, manual rollback |
| Workflow ID / Run ID | Idempotency key or Workflow ID                                                                                     | Seen-ID set for deduplication |
| Cron / Scheduled trigger | Delayed self-invocation                                                                                            | External cron, setInterval, CloudWatch Events |
| Wait for result of background job | `ctx.attach(invocationId)`                                                                                         | State polling, callback webhook |

### Key differences from orchestrators

- **No separate "worker" concept.** Services are HTTP servers (or Lambda functions). You register them in Restate and Restate then routes to them. No worker pools, task queues, or polling.
- **No workflow definition DSL.** Logic is plain code with `ctx` calls. No YAML, JSON, or visual workflow definitions.
- **State lives in Virtual Objects, not workflow variables.** For entity state that outlives a single process, use Virtual Objects. Workflow state is scoped to one execution.
- **Restate Workflows are for exactly-once-per-ID processes.** For reusable orchestration logic that runs multiple times with the same parameters, use a Service handler instead.

## General application mapping

Map common application components to Restate service types. Use this table to systematically convert each component of the existing architecture.

| Existing component                              | Restate equivalent                                                                 | Rationale                                                                                             |
|-------------------------------------------------|------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| REST API handlers (stateless)                   | Service handlers                                                                   | Each request is independent. No shared state.                                                         |
| Stateful entities (users, carts, sessions)      | Virtual Objects keyed by entity ID                                                 | Persistent K/V state per entity with single-writer consistency.                                       |
| Relational database                             | Keep it + `ctx.run()` database calls: https://docs.restate.dev/guides/databases.md | Complex access patterns. Full SQL. Access from other services.                                        |
| Multi-step processes, approval flows            | Workflows                                                                          | Exactly-once execution per ID. Durable promises for human-in-the-loop.                                |
| Background/cron jobs                            | Delayed self-invocation                                                            | Handler sends a delayed message to itself or another handler. Durable timer.                          |
| Message queue consumers (Kafka, SQS, RabbitMQ)  | Kafka ingestion or one-way sends                                                   | Restate has native Kafka integration. Alternatively, push events via one-way sends.                   |
| External API calls (payments, emails, webhooks) | Wrap in `ctx.run()`                                                                | Journaled for durability. Not re-executed on replay.                                                  |
| In-memory caches                                | Virtual Object state                                                               | Persistent, survives restarts. Scoped per key.                                                        |
| Workflow engines / state machines               | Workflows (linear) or Virtual Objects (ongoing)                                    | Workflow for processes with start/end. Virtual Object for long-lived entities with state transitions. |
| Scheduled tasks (cron, delayed jobs)            | Delayed sends                                                                      | `ctx.send()` with delay parameter. Survives crashes.                                                  |
| Distributed locks                               | Virtual Object exclusive handlers                                                  | Single-writer guarantee per key acts as a durable lock. No external lock service needed.              |
| Retry logic with exponential backoff            | Restate default retry + service/handler/`ctx.run()` options                        | Remove manual retry wrappers. Restate retries automatically. Configure per-operation limits.          |

### Migration steps

1. **Prioritize.** Identify the highest-value component to migrate first. Good candidates: the orchestrator, the most failure-prone/long-running component, the one with the most manual retry/compensation logic.
2. **Map to service types.** Choose Service, Virtual Object, or Workflow for each component using the table above. Present the architecture to the user (see template below).
3. **Wrap side effects.** Place all non-deterministic operations (API calls, DB writes, external HTTP requests) inside `ctx.run()`.
4. **Replace retry logic.** Remove manual retry wrappers, circuit breakers, and exponential backoff code. Restate retries automatically. Use `TerminalError` for permanent failures.
5. **Migrate state.** Replace in-memory or kv-store-backed entity state with Virtual Object K/V state where it simplifies the architecture. Keep external databases for data that needs SQL queries, complex indexes, or is shared outside Restate.
6. **Replace communication infrastructure.** Remove message queue producers/consumers, HTTP client retry logic, and polling loops. Use durable RPC, one-way sends, or delayed sends.
7. **Register and test.** Register the service with `restate deployments register`. Test via curl or the Restate UI (port 9070). Verify journal entries appear in the UI.
8. **Iterate.** Gradually migrate remaining components. Migrated and non-migrated services can coexist: non-Restate services call Restate services via HTTP, Restate services call external services via `ctx.run()`.

## Architecture presentation template

After analyzing the existing codebase, present the proposed Restate architecture to the user for approval before proceeding to implementation.

```
Proposed Restate Architecture:

Services:
- [ComponentName] -> [Service | Virtual Object (keyed by X) | Workflow] - [rationale]
- ...

Communication:
- [How services interact: durable RPC, one-way sends, delayed messages, awakeables]

State:
- [What moves to Restate K/V state vs what stays in external databases]
- [Rationale for each decision]

Migration order:
- [Which components to migrate first and why]
```

Wait for user approval before proceeding to implementation. Adjust the architecture based on user feedback.

## What stays outside Restate

Not everything should move into Restate. Keep the following external:

- **Relational data with complex queries**: SQL databases for data that needs joins, aggregations, full-text search, or is accessed by non-Restate systems. Use `ctx.run()` to wrap database calls for durability.
- **Large binary data**: Files, images, videos. Store in object storage (S3, GCS). Store references (URLs, keys) in Virtual Object state, not the data itself.
- **Authentication/authorization**: Keep existing auth middleware. Restate services sit behind the auth layer. Do not move auth logic into handlers.
- **Existing APIs consumed by external clients**: Do not force external consumers to change. Place a Restate service behind the existing API gateway or call Restate via HTTP from the existing API layer.
- **Analytics and reporting databases**: Data warehouses, OLAP stores, and analytics pipelines. Restate K/V state is for operational state, not analytical queries.
- **Third-party managed services**: Payment gateways, email providers, SMS services. Wrap calls to these in `ctx.run()` but do not try to replace them.

## Common migration pitfalls

- **Over-migrating**: Not every component benefits from durable execution. 
- **Ignoring the call graph**: Migrating Virtual Objects without checking for exclusive handler cycles leads to deadlocks. Map the call graph first (see `references/design-and-architecture.md`).
- **Wrapping entire handlers in a single `ctx.run()`**: This defeats the purpose. Each independent side effect should be its own `ctx.run()` block so Restate can skip already-completed steps on replay.
- **Keeping external retry logic**: Remove existing retry wrappers, circuit breakers, and exponential backoff code. Letting both Restate and application-level retries run creates redundant retries and unexpected behavior.
- **Large payloads in K/V state**: Virtual Object state is for operational data (kilobytes). Store large payloads in external storage and keep references in state.

## Further reading

For Virtual Object keying strategy, handler design (exclusive vs shared), deadlock prevention, and common architecture patterns, load `references/design-and-architecture.md`.
