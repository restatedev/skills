# Interacting with Restate Services

How to invoke services, interact with running invocations, and control their lifecycle. These are application-level concerns, not just operational tools. For example: a chatbot cancels an in-flight agent when the user sends a new message, a frontend attaches to a running workflow to retrieve the result, or a webhook uses idempotency keys to deduplicate retries.

## Invoking services over HTTP

All Restate services are reachable via HTTP through the Restate ingress (default port 8080).

```bash
# Service handler
curl localhost:8080/MyService/myHandler --json '"input"'

# Virtual Object handler (keyed by entity ID)
curl localhost:8080/MyObject/myKey/myHandler --json '"input"'

# Workflow run handler (keyed by workflow ID)
curl localhost:8080/MyWorkflow/myWorkflowId/run --json '"input"'

# Fire-and-forget (returns immediately, invocation runs in background)
curl localhost:8080/MyService/myHandler/send --json '"input"'

# Delayed invocation (executes after delay)
curl "localhost:8080/MyService/myHandler/send?delay=10s" --json '"input"'

# With idempotency key (deduplicates for 24h by default)
curl localhost:8080/MyService/myHandler -H 'idempotency-key: my-unique-key' --json '"input"'
```

For the OpenAPI spec of registered services: `curl localhost:9070/services/MyService/openapi`

## Invoking via SDK clients

Each SDK provides a typed client for invoking services from external code (outside Restate handlers). See the SDK-specific `api-and-pitfalls.md` for client patterns. For the full client API, use the bundled MCP server.

Common pattern: a web server or frontend calls Restate services via the SDK client or HTTP, then uses attach or polling to get results.

## Invoking via Kafka

Restate can consume Kafka topics and route messages to handlers. Configure subscriptions via the Admin API or CLI. Each Kafka message becomes a Restate invocation with exactly-once processing guarantees.

For Kafka setup and configuration: use the bundled MCP server or see [Kafka invocations](https://docs.restate.dev/services/invocation/kafka).

## Interacting with invocations

These primitives are part of application logic, not just operations.

| I need to... | Use | Not                     | Example use case |
|---|---|-------------------------|---|
| Deduplicate external triggers | Idempotency key on send/call | Seen-ID set in state    | Webhook retries, user double-clicks |
| Wait for a background result | Attach to the invocation (SDK-specific API) | State polling, callback | Frontend waiting for agent response |
| Cancel a running invocation | SDK cancel API or Admin API | Cancel flag + polling   | Chatbot: interrupt agent on new message |
| Bound how long to wait | `.orTimeout({ seconds: N })` | /                       | Deadline on a service call |
| Pause for external event | Awakeable or Durable Promise | Polling loop            | Human approval, payment callback |
| Schedule future work | Delayed send | `ctx.sleep()` + send    | Reminders, retry-after, scheduled tasks |
| Periodic execution | Delayed self-invocation | External cron           | Polling external API, periodic cleanup |

For detailed SDK-specific API: see the `api-and-pitfalls.md` reference for the detected SDK.

Docs:
- [Service communication](https://docs.restate.dev/develop/ts/service-communication.md) - Durable RPC, sends, delays
- [External events](https://docs.restate.dev/develop/ts/external-events.md) - Awakeables, durable promises
- [Managing invocations](https://docs.restate.dev/services/invocation/managing-invocations.md) - Cancel, kill, pause, resume, attach
- [Idempotent invocations](https://docs.restate.dev/foundations/invocations.md) - Deduplication

## Retention

| What | Default | Config |
|------|---------|--------|
| Virtual Object K/V state | Indefinite (until cleared) | Manual `ctx.clear()` |
| Workflow K/V state | 24h after `run` completes | `workflowRetention` |
| Idempotency keys | 24h | `idempotencyRetention` |
| Invocation journals | 24h after completion | `journalRetention` |

After workflow retention expires, shared handlers return errors and state is inaccessible. Mirror critical results to a Virtual Object before the workflow completes.

Docs: [Service configuration](https://docs.restate.dev/services/configuration.md)

## CLI quick reference

```bash
restate deployments register http://localhost:9080 [--force]
restate deployments list
restate invocations list [--status backing-off] [--service MyService]
restate invocations describe <id>
restate invocations cancel <id|service|service/handler>
restate invocations kill <id>          # Last resort, no compensation
restate invocations pause <id>
restate invocations resume <id>
restate sql "SELECT * FROM sys_invocation WHERE target_service_name = 'MyService'"
restate kv get <SERVICE> <KEY>
```

## Key documentation

- [Invocations overview](https://docs.restate.dev/foundations/invocations) - Invocation model, idempotency, lifecycle
- [Service communication](https://docs.restate.dev/develop/ts/service-communication) - Durable RPC, sends, delays, attach
- [Managing invocations](https://docs.restate.dev/services/invocation/managing-invocations) - Cancel, kill, pause, resume, purge
- [Kafka invocations](https://docs.restate.dev/services/invocation/kafka) - Event-driven ingestion
- [Service configuration](https://docs.restate.dev/services/configuration) - Timeouts, retention, retry policies
- [Admin API](https://docs.restate.dev/references/admin-api) - OpenAPI spec for programmatic control
- [SDK clients](https://docs.restate.dev/services/invocation/clients/) - Typed client invocations per SDK
