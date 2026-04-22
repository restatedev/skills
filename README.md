# Restate Coding Agent Plugin

> This is an early preview of the Restate Coding Agent Plugin. Please submit feedback (positive and negative) in the [Restate Discord](https://discord.restate.dev), [Slack Community](https://slack.restate.dev), or via issues in this repo.

A coding agent plugin for building applications with [Restate](https://restate.dev) — the durable execution runtime for resilient services, workflows, and AI agents.

Packaged for **Claude Code**, **Cursor**, and **Codex**. Works across four Restate SDKs: **TypeScript**, **Python**, **Java**, and **Go**.

## What this plugin provides

- **One skill** (`restate`) that activates automatically when you mention Restate concepts or open a Restate template. It detects your SDK from `package.json`, `requirements.txt`/`pyproject.toml`, `pom.xml`/`build.gradle`, or `go.mod` and loads the right reference on demand.
- **One MCP server** (`restate-docs`) bound to `https://docs.restate.dev/mcp` for searching conceptual guides, deployment docs, server config, and advanced topics.

## Install

The same plugin source ships manifests for Claude Code, Cursor, and Codex — all three live under `plugins/restatedev/` and share one `skills/` tree.

### Claude Code

Add the marketplace and install the plugin from within Claude Code:

```
/plugin marketplace add restatedev/skills
/plugin install restatedev@restatedev-skills
```

To verify, run `/plugin` — you should see `restatedev-plugin` listed as enabled.

### Cursor

Cursor discovers the plugin via `.cursor-plugin/marketplace.json` at the repo root. Follow Cursor's plugin marketplace workflow to install from `restatedev/skills`.

### Codex CLI

Codex CLI reads the manifest at `plugins/restatedev/.codex-plugin/plugin.json`. Follow the Codex CLI plugin install flow to add `restatedev/skills`.

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

Paths above are relative to `plugins/restatedev/skills/building-restate-services/`.

Deployment, server configuration, Kafka setup, and other advanced topics are handled by querying the bundled `restate-docs` MCP server.

## Contributing

Test the Claude Code plugin locally with:
```shell
claude --plugin-dir /path/to/this/repo/plugins/restatedev
```

For Cursor and Codex CLI, point their respective local-plugin flags at the same `plugins/restatedev/` directory.

> **Note:** The `plugins/restatedev/skills/` folder in this repo is synced automatically every 12 hours from [`restatedev/docs-restate`](https://github.com/restatedev/docs-restate/tree/main/restate-plugin/src), where the skills are authored, tested, and updated. Edits made directly here will be overwritten — contribute changes upstream in `docs-restate` instead. Reference files live under `plugins/restatedev/skills/building-restate-services/references/`. When adding a new topic, link it from `SKILL.md`'s context table so the skill knows when to load it.

## Links

- Restate homepage: https://restate.dev
- Documentation: https://docs.restate.dev
- Service examples: https://github.com/restatedev/examples
- AI agent examples: https://github.com/restatedev/ai-examples
- SDK repos: [sdk-typescript](https://github.com/restatedev/sdk-typescript), [sdk-python](https://github.com/restatedev/sdk-python), [sdk-java](https://github.com/restatedev/sdk-java), [sdk-go](https://github.com/restatedev/sdk-go)

## License

MIT
