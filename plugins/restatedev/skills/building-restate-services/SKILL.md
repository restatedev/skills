---
name: building-restate-services
description: >
  Design, build, run, and test Restate durable services, virtual objects,
  workflows, and AI agents across TypeScript, Python, Java, and Go.
  This skill should be used when the user mentions "restate", "durable execution",
  "virtual object", "restate service", "restate workflow", or "durable agent"
  or wants to build resilient backend services, AI agents, or workflows
  with automatic failure recovery. Also use when converting existing applications
  or migrating from workflow orchestrators to Restate.
  Use proactively when a project contains restate dependencies in
  package.json, requirements.txt, pyproject.toml, pom.xml, build.gradle, or go.mod.
---

# Restate

Restate is a durable execution runtime. It records every step of a handler in a journal, so if the process crashes, the handler replays from the journal and resumes exactly where it left off. Services are regular applications using the Restate SDK (TypeScript, Python, Java, Go) that run behind a Restate Server (a single Rust binary on ports 8080 for ingress and 9070 for admin UI).

## Detect context

Scan the project to determine the SDK and context:

1. **Detect SDK**:
   - `package.json` with `@restatedev/restate-sdk` -> TypeScript
   - `requirements.txt` or `pyproject.toml` with `restate-sdk` -> Python
   - `pom.xml` or `build.gradle` with `dev.restate` -> Java
   - `go.mod` with `github.com/restatedev/sdk-go` -> Go
   - If no SDK detected, check for project files to determine language, then ask the user

2. **Detect existing Restate code**: Grep for `@restatedev`, `restate-sdk`, `dev.restate.sdk`, `github.com/restatedev`

3. **Detect AI frameworks**: `@ai-sdk/`, `openai-agents`, `google-adk`, `pydantic-ai`

4. **Detect workflow orchestrators**: `temporalio`, `@temporalio`, `aws-cdk/aws-stepfunctions`, etc.

## Core reference (always load)

After detecting the SDK, always load the SDK reference:

- `references/<sdk>/api-and-pitfalls.md` - Setup, API patterns, determinism rules, error handling, and pitfalls for the detected SDK

## Context-based references (load when relevant)

| Context | Reference |
|---------|-----------|
| Design a new application, choose service types | `references/design-and-architecture.md` |
| Convert from a workflow orchestrator or existing app | `references/translate-to-restate.md` |
| Invoking services, interacting with invocations (cancel, attach, idempotency, sends, Kafka) | `references/invocation-lifecycle.md` |
| Build AI agent with Vercel AI SDK | `references/ts/restate-vercel-ai-agents.md` |
| Build AI agent with OpenAI Agents SDK | `references/python/restate-openai-agents-agents.md` |
| Build AI agent with Google ADK | `references/python/restate-google-adk-agents.md` |
| Build AI agent with Pydantic AI | `references/python/restate-pydantic-ai-agents.md` |
| Debug errors, stuck invocations, journal mismatches | `references/debug-applications.md` |
| Testing, deployment, server config, Kafka, advanced topics | Use the bundled **restate-docs** MCP server |
| Code examples and templates | `github.com/restatedev/examples`, `github.com/restatedev/ai-examples` |

## Before-you-design checklist

Before designing any Restate service architecture, check:

| Question                                                          | Consult |
|-------------------------------------------------------------------|---------|
| Restate service types, stateful entities, keying, concurrency?    | `references/design-and-architecture.md` |
| Non-Restate orchestrator in the project?                          | `references/translate-to-restate.md` |
| Invoking, cancelling, deduplicating, or attaching to invocations? | `references/invocation-lifecycle.md` |
| Error handling, compensation, sagas?                              | [Error handling guide](https://docs.restate.dev/guides/error-handling), [Sagas guide](https://docs.restate.dev/guides/sagas) |
| AI agent or LLM calls?                                            | The relevant agent integration reference |

## Always verify before finishing

**All checks below are mandatory**.

- [ ] All side effects, external I/O, and DB calls wrapped in `ctx.run()`
- [ ] No native random, time, sleep, or UUID -- use ctx helpers
- [ ] Restate concurrency combinators only (no `Promise.all`, `asyncio.gather`/`wait`, `CompletableFuture`, goroutines + channels, `select`)
- [ ] No ctx operations inside `ctx.run()`
- [ ] `TerminalError` raised for non-retryable failures
- [ ] Python: no bare `except:`
- [ ] AI agents: set a retry policy for LLM calls
- [ ] Virtual Objects: no deadlock cycles
- [ ] Service registered and invoked via curl or the UI

### Replay test - required on any handler logic change

Any change to handler business logic (new `ctx` operations, reordered steps, new `ctx.run()` blocks, new branches) must be covered by a Testcontainers test with **always-replay** enabled, and that test must pass before declaring the work done. Always-replay forces every journaled step to replay on every invocation, so non-determinism fails the test instead of failing in production on retry.

- TypeScript: `alwaysReplay: true` in `RestateTestEnvironment.start`
- Python: `always_replay=True` in `restate.test_harness`
- Java / Go: `RESTATE_WORKER__INVOKER__INACTIVITY_TIMEOUT=0m` on the Restate container

See the Testing section of `references/<sdk>/api-and-pitfalls.md` for the working scaffold.
