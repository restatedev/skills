# Debug Reference

## Common Errors and Fixes

### Journal mismatch (RT0016)

Code changed while an invocation was in flight. The journal no longer matches the code.

**Safe changes:** Fixing bugs inside `ctx.run()` (the side effect name and position remain the same).

**Unsafe changes:** Reordering, adding, or removing Restate operations (calls, sleeps, `ctx.run`, etc.).

Read https://docs.restate.dev/services/versioning

### Service not found / handler not found (RT0011, RT0018)

The service is not registered, or the handler name does not match.

**Fix:** Register the deployment and verify names match:

```bash
restate deployments register http://localhost:9080
restate services list
```

### Invocation stuck / timeout (RT0001)

A long-running `ctx.run()` without journal entries triggers the inactivity timeout (default 1 minute).

**Fix:** Increase the inactivity timeout in the service configuration. For LLM calls, set to 3 to 5 minutes.

### Discovery fails (META0003)

Restate cannot reach the service endpoint.

**Fix:** Check the URI, ensure the service is running, and verify network connectivity. When running Restate in Docker, use `http://host.docker.internal:9080` instead of `localhost`.

### Deployment conflict (META0004)

The same URI is already registered with different services or handlers.

**Fix:** Use the `--force` flag to override: `restate deployments register --force http://localhost:9080`

### HTTP/1.1 required (META0014)

The service responds using HTTP/1.1 instead of HTTP/2.

**Fix:** Use the `--use-http1.1` flag: `restate deployments register --use-http1.1 http://localhost:9080`

### State not persisting

Using a Service instead of a Virtual Object. Services are stateless and have no persistent state.

**Fix:** Switch to a Virtual Object (`restate.object` in TypeScript, `restate.VirtualObject` in Python).

### Workflow resubmission fails

Workflows run exactly once per ID. Submitting the same ID again attaches to the existing execution.

**Fix:** Use a new workflow ID, or check the existing workflow status.

### Deadlocked Virtual Objects

Exclusive handlers on the same Virtual Object key calling each other cause deadlocks.

**Fix:** Cancel the stuck invocation, then refactor to use one-way sends instead of request-response calls between exclusive handlers on the same key:

```bash
restate invocations cancel <id>
```

---

## Essential CLI Commands

```bash
# Deployment management
restate deployments register http://localhost:9080 [--force]
restate deployments list

# Invocation inspection
restate invocations list [--status backing-off] [--service MyService]
restate invocations describe <id>

# Invocation lifecycle
restate invocations cancel <id|service|service/handler>
restate invocations kill <id>
restate invocations pause <id>
restate invocations resume <id> [--deployment latest]

# SQL introspection
restate sql --json "SELECT * FROM sys_invocation WHERE target_service_name = 'MyService'"

# State inspection
restate kv get <SERVICE> <KEY>
restate kv edit <SERVICE> <KEY>
```

---

## Admin API

Base URL: `http://localhost:9070/openapi`
