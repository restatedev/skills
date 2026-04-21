# Restate Coding Agent Plugin

A coding agent plugin for building applications with [Restate](https://restate.dev) — the durable execution runtime for resilient services, workflows, and AI agents.

Works across four Restate SDKs: **TypeScript**, **Python**, **Java**, and **Go**.

## What this plugin provides

- **One skill** (`restate`) that activates automatically when you mention Restate concepts or open a Restate template. It detects your SDK from `package.json`, `requirements.txt`/`pyproject.toml`, `pom.xml`/`build.gradle`, or `go.mod` and loads the right reference on demand.
- **One MCP server** (`restate-docs`) bound to `https://docs.restate.dev/mcp` for searching conceptual guides, deployment docs, server config, and advanced topics.

## Install

Add the marketplace and install the plugin from within Claude Code:

```
/plugin marketplace add restatedev/skills
/plugin install restatedev@restatedev-skills
```

To verify, run `/plugin` — you should see `restatedev-plugin` listed as enabled.

## What it helps with

The skill triggers on mentions of Restate, durable execution, virtual objects, workflows, durable agents, and related terms.

Once active, it progressively loads references for:

| Topic                                                                                        | Reference |
|----------------------------------------------------------------------------------------------|---|
| SDK setup, API patterns, determinism rules, error handling, testing, pitfalls                | `references/<sdk>/api-and-pitfalls.md` |
| Designing a new Restate application, picking service types                                   | `references/design-and-architecture.md` |
| Migrating to Restate from other workflow orchestrators or general microservices applications | `references/translate-to-restate.md` |
| Invoking, cancelling, attaching, idempotency, sends, Kafka                                   | `references/invocation-lifecycle.md` |
| Debugging stuck invocations and journal mismatches                                           | `references/debug-applications.md` |
| AI agents with Vercel AI SDK (TS)                                                            | `references/ts/restate-vercel-ai-agents.md` |
| AI agents with OpenAI Agents SDK (Python)                                                    | `references/python/restate-openai-agents-agents.md` |
| AI agents with Google ADK (Python)                                                           | `references/python/restate-google-adk-agents.md` |
| AI agents with Pydantic AI (Python)                                                          | `references/python/restate-pydantic-ai-agents.md` |

Deployment, server configuration, Kafka setup, and other advanced topics are handled by querying the bundled `restate-docs` MCP server.

## Contributing

Test a plugin locally with:
```shell
claude --plugin-dir /path/to/this/repo
```

Reference files live under `skills/building-restate-services/references/`. When adding a new topic, link it from `SKILL.md`'s context table so the skill knows when to load it.

## Links

- Restate homepage: https://restate.dev
- Documentation: https://docs.restate.dev
- Service examples: https://github.com/restatedev/examples
- AI agent examples: https://github.com/restatedev/ai-examples
- SDK repos: [sdk-typescript](https://github.com/restatedev/sdk-typescript), [sdk-python](https://github.com/restatedev/sdk-python), [sdk-java](https://github.com/restatedev/sdk-java), [sdk-go](https://github.com/restatedev/sdk-go)

## License

MIT
