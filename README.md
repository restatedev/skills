# Restate Coding Agent Plugin and Skills

> This is an early preview of the Restate Coding Agent Plugin. Please submit feedback (positive and negative) in the [Restate Discord](https://discord.restate.dev), [Slack Community](https://slack.restate.dev), or via issues in this repo.

A coding agent plugin for building applications with [Restate](https://restate.dev) — the durable execution runtime for resilient services, workflows, and AI agents.

Packaged for **Claude Code**, **Cursor**, and **Codex**. Works across four Restate SDKs: **TypeScript**, **Python**, **Java**, and **Go**.

## What this plugin provides

- **Skill**: `building-restate-services` — activates automatically when you mention Restate concepts or open a Restate template. Detects your SDK from `package.json`, `requirements.txt`/`pyproject.toml`, `pom.xml`/`build.gradle`, or `go.mod` and loads the right reference on demand.
- **One MCP server** (`restate-docs`) bound to `https://docs.restate.dev/mcp` for searching conceptual guides, deployment docs, server config, and advanced topics.

## Install Plugin

The same plugin source ships manifests for Claude Code, Cursor, and Codex — all three live under `plugins/restatedev/` and share one `skills/` tree.

### Claude Code

Add the marketplace and install the plugin from within Claude Code:

```
/plugin marketplace add restatedev/skills
/plugin install restatedev@restatedev-plugin
```

`/plugin install` opens the plugin details, where you choose the installation scope. To verify, run `/plugin` — you should see `restatedev-plugin` listed as enabled.

Claude Code does not auto-update third-party marketplaces by default. To receive skill updates automatically, enable auto-update for `restatedev-plugin` in the **Marketplaces** tab of `/plugin`.

### Cursor

Cursor discovers the plugin via `.cursor-plugin/marketplace.json` at the repo root.

1. On a Teams or Enterprise plan, an admin imports the marketplace from the Cursor dashboard: **Plugins & MCPs** → **Import from Repo**, with `https://github.com/restatedev/skills` (use the GitHub URL, not a local folder path).
2. Open **Customize** in the Cursor sidebar, install the **restatedev** plugin, and choose the project or user scope.

This plugin is not listed in the public [Cursor Marketplace](https://cursor.com/marketplace) search yet. For local development, copy `plugins/restatedev/` into `~/.cursor/plugins/local/restatedev/` (real files; symlinks outside that directory are rejected) and run **Developer: Reload Window**. On Enterprise plans, an admin must enable **Allow Local Plugin Imports** first.

### Codex

Add the marketplace: 
```shell
codex plugin marketplace add restatedev/skills
```

Install the plugin through Codex's TUI: launch `codex`, run `/plugins`, find the Restate marketplace, select the Restate plugin, and choose Install. Start a new Codex session after the installation completes.

### Other coding agents

To add the Restate skills to any coding agent:

```shell
npx skills add restatedev/skills
```

You will need to add the MCP server manually in this case. See [`.mcp.json`](plugins/restatedev/.mcp.json).

## What it helps with

The `building-restate-services` skill triggers on mentions of Restate, durable execution, virtual objects, workflows, durable agents, and related terms.

Once active, it progressively loads references for:

| Topic                                                                                        | Reference |
|----------------------------------------------------------------------------------------------|---|
| SDK setup, API patterns, determinism rules, error handling, testing, pitfalls                | `references/<sdk>/api-and-pitfalls.md` |
| Durable Coroutines with the TypeScript Gen SDK                                               | `references/ts/durable-coroutines.md` |
| Designing a new Restate application, picking service types                                   | `references/design-and-architecture.md` |
| Concurrency limits, multi-tenant fairness, cost controls, scopes, limit keys                 | `references/flow-control-and-scopes.md` |
| Migrating to Restate from other workflow orchestrators or general microservices applications | `references/translate-to-restate.md` |
| Invoking, cancelling, attaching, idempotency, sends, Kafka                                   | `references/invocation-lifecycle.md` |
| Debugging stuck invocations and journal mismatches                                           | `references/debug-applications.md` |
| Designing AI agents, choosing a durability boundary, unsupported LLM or agent SDKs           | `references/ai-agents.md` |
| AI agents with Vercel AI SDK (TS)                                                            | `references/ts/restate-vercel-ai-agents.md` |
| AI agents with OpenAI Agents SDK (Python)                                                    | `references/python/restate-openai-agents-agents.md` |
| AI agents with Google ADK (Python)                                                           | `references/python/restate-google-adk-agents.md` |
| AI agents with Pydantic AI (Python)                                                          | `references/python/restate-pydantic-ai-agents.md` |
| AI agents with LangChain or LangGraph (Python)                                               | `references/python/restate-langchain-agents.md` |

Paths above are relative to `plugins/restatedev/skills/building-restate-services/`.

The skill also instructs the agent to use the `restate` CLI (version 1.8 or later) to register deployments, inspect and manage invocations, read and edit state, and query the server with SQL.

For deployment, server configuration, Kafka setup, and other advanced topics, the agent queries the bundled `restate-docs` MCP server.

## Contributing

Test the Claude Code plugin locally with:
```shell
claude --plugin-dir /path/to/this/repo/plugins/restatedev
```

For Cursor and Codex CLI, point their respective local-plugin flags at the same `plugins/restatedev/` directory.

> **Note:** The skills are authored and tested in [`restatedev/docs-restate`](https://github.com/restatedev/docs-restate/tree/main/restate-plugin/src) under `restate-plugin/src/`. A generator produces `restate-plugin/skills/` from those sources, and a GitHub Actions workflow copies that folder into `plugins/restatedev/skills/` in this repo every 12 hours. Edits made directly here will be overwritten — contribute changes to `restate-plugin/src/` in `docs-restate` instead. Reference files live under `plugins/restatedev/skills/building-restate-services/references/`. When adding a new topic, link it from `SKILL.md`'s context table so the skill knows when to load it.

## Links

- Restate homepage: https://restate.dev
- Documentation: https://docs.restate.dev
- Service examples: https://github.com/restatedev/examples
- AI agent examples: https://github.com/restatedev/ai-examples
- SDK repos: [sdk-typescript](https://github.com/restatedev/sdk-typescript), [sdk-python](https://github.com/restatedev/sdk-python), [sdk-java](https://github.com/restatedev/sdk-java), [sdk-go](https://github.com/restatedev/sdk-go)

## License

MIT
