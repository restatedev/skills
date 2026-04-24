# AI agent on-ramp

For users who want to build an AI agent on Restate. Unlike the Workflows / Orchestration tours, there is **no single `restate example` template** — the agent frameworks differ enough that we route per framework.

## Intake follow-up

Ask one question with `AskUserQuestion`:

*"Which agent framework would you like to use?"*

- **Vercel AI SDK** (TypeScript) — streaming, tool calls, works well with Edge/Workers.
- **OpenAI Agents SDK** (Python) — OpenAI-first, structured outputs, handoffs.
- **Google ADK** (Python) — Google Agent Development Kit.
- **Pydantic AI** (Python) — type-safe, provider-agnostic.
- **Not sure / just Restate concepts first** — run the Quickstart (`quickstart.md`) instead, then come back here.

## Why Restate for agents

Give this in 3–4 bullets before cloning anything:

- LLM calls are slow and flaky — Restate retries them durably without re-running earlier tool calls.
- Long-running agents (days/weeks) survive process restarts because state and journal are persisted.
- Human-in-the-loop is trivial: `ctx.awakeable()` blocks until someone resolves it.
- Tool calls + compensations + timers are all primitives the agent can just use.

## Source examples

Clone one of these — no CLI template yet:

```bash
git clone https://github.com/restatedev/ai-examples
cd ai-examples
```

Browse to the directory matching the framework:

- `ai-examples/typescript/vercel-ai-sdk/...`
- `ai-examples/python/openai-agents/...`
- `ai-examples/python/google-adk/...`
- `ai-examples/python/pydantic-ai/...`

Pick one with a focused scope (a single agent + 1–2 tools). Read its `README.md` to run it.

## Reference loading

After cloning, load the framework-specific reference from the sibling `building-restate-services` skill:

| Framework | Reference |
|---|---|
| Vercel AI SDK | `plugins/restatedev/skills/building-restate-services/references/ts/restate-vercel-ai-agents.md` |
| OpenAI Agents | `plugins/restatedev/skills/building-restate-services/references/python/restate-openai-agents-agents.md` |
| Google ADK | `plugins/restatedev/skills/building-restate-services/references/python/restate-google-adk-agents.md` |
| Pydantic AI | `plugins/restatedev/skills/building-restate-services/references/python/restate-pydantic-ai-agents.md` |

These references are the canonical guides for wiring each framework into a Restate handler. Read the relevant one before running the example.

## Walkthrough shape

For the chosen example, use the same stage pattern as the other tours:

1. **Read the agent handler** — show how the framework's agent loop sits inside a Restate handler. Highlight where tool calls happen and how they're wrapped.
2. **Run the service** — `npm run dev` / `uv run ...` as appropriate.
3. **Register and invoke** — `restate deployments register ...`, then curl/playground to kick off the agent.
4. **Watch the journal** — in the admin UI, open the invocation and scroll the journal. The user should see each LLM call and each tool call as a separate entry.
5. **Crash-and-resume demo** — kill the service mid-agent-step, restart, watch it resume without re-running completed tool calls (this is the headline feature for agents).
6. **Human-in-the-loop demo (if the example has one)** — invoke, grab the awakeable id, resolve it with curl, watch the agent continue.

## Handoff

- *"Ready to build your own agent?"* → hand off to the sibling `building-restate-services` skill; it already has the framework-specific guidance.
- *"Want to first nail the Restate fundamentals?"* → Quickstart (`quickstart.md`) or Tour of Workflows (`tour-of-workflows.md`).
- Deeper framework questions → `restate-docs` MCP server + the framework's own docs.
