---
name: learn-restate-by-example
description: >
  Interactive, hands-on introduction to Restate tailored to the user's use case.
  Use this skill when the user wants to learn, explore, or try out Restate for
  the first time: phrases like "teach me restate", "learn restate", "getting
  started with restate", "restate tutorial", "restate quickstart", "restate
  tour", "restate by example", "show me restate", "I want to explore restate",
  or "evaluate restate for my project". The skill picks the right learning
  track (Quickstart, Tour of Workflows, Tour of Microservice Orchestration, or
  AI agent on-ramp) based on what the user is trying to build, clones the
  matching starter repo, walks through the stages live while running commands
  locally, and hands off to the building-restate-services skill when the user
  is ready to build their own application.
---

# Learn Restate by example

This is the **onboarding** skill for Restate. It picks a track based on what the user is trying to build, clones the matching starter, and walks through the stages hands-on in the user's terminal.

The sibling `building-restate-services` skill is the *reference* tool for users actively writing code. This skill hands off to it once the learner is ready.

## When to fire

Fire **only on learning intent**. Signals:

- The user explicitly asks to learn, explore, evaluate, or be shown Restate.
- The user is in an empty or near-empty directory and mentions Restate.
- The user says they are new to Restate or asks "how do I get started with Restate".

Do **not** fire when:

- The user is asking a reference question about Restate APIs, errors, or patterns (e.g. "how do I use `ctx.run`?"). That's the sibling `building-restate-services` skill.
- The user already has a Restate project and is adding to it. Unless they explicitly ask for a walkthrough, defer to the sibling skill.

If unclear, ask one short clarifying question before firing.

## Flow

1. **Intake** — ask two questions with `AskUserQuestion`:
   1. *What are you trying to build?* — options: long-running workflow / coordinating multiple services / AI agent / just want to see it work / not sure.
   2. *Which language?* — TypeScript / Python / Java / Go.

   If "not sure", ask a free-form follow-up about their actual problem, then route per `references/tracks.md`.

2. **Route to a track** (load the matching reference):

   | User intent | Track | Reference |
   |---|---|---|
   | Just show me / evaluate / hello world | Quickstart | `references/quickstart.md` |
   | Long-running workflow, signup, order processing, background jobs, retries, timers, sagas on one workflow | Tour of Workflows | `references/tour-of-workflows.md` |
   | Coordinating multiple services, virtual objects, sagas across services, idempotency between services | Tour of Microservice Orchestration | `references/tour-of-orchestration.md` |
   | AI agent, LLM calls, tool-using agent, agent with memory | AI agent on-ramp | `references/ai-agents.md` |

   If the user's intent maps to more than one track, prefer the simpler one first (Quickstart > Tour of Workflows > Tour of Orchestration) and offer the other as a follow-up.

3. **Check prerequisites** — load `references/prerequisites.md`, run detection, offer to install anything missing. Ask before running install commands.

4. **Scaffold the starter** — run `restate example <template>` in the user's CWD. The template names are in each track's reference file.

5. **Start the stack** — start the Restate server (Docker or native binary, whichever the user has) and the service from the starter. Wait for the service to register. Confirm by listing services via the CLI or the admin UI at `http://localhost:9070`.

6. **Walk through stages** — one stage at a time, per the track's reference:
   - Explain the concept in 2–4 lines.
   - Point at the specific file/lines in the cloned starter (read them, don't paste the doc).
   - Run the exact curl or CLI command.
   - Describe what the user should observe.
   - Pause for questions; offer to continue, jump to a specific stage, or diverge.

7. **Handoff** — when the track ends:
   - Recommend the sibling `building-restate-services` skill for actual implementation.
   - Point at the bundled `restate-docs` MCP server for deployment, Kafka, server config, and other advanced topics.
   - Point at `github.com/restatedev/examples` and `github.com/restatedev/ai-examples` for more code.

## Ground rules during the walkthrough

- **Run one command at a time.** Don't batch curl calls — the user is learning, give them time to look at each response.
- **Read the cloned starter files.** Don't reinvent or paraphrase code that already exists in the repo. Use line-number citations.
- **Keep the server running.** If the service crashes or is stopped, restart it — do not start over.
- **Don't re-explain the docs.** The stage notes are deliberately terse paraphrases; if the user wants depth, point at the canonical URL in the stage reference.
- **Let the user deviate.** If they ask a side question, answer it using the sibling skill's references or the `restate-docs` MCP server, then offer to resume the track.
- **Respect permissions.** Installs, `docker run`, and server starts are not silent — explain what will happen and get consent.

## Stage reference files

- `references/tracks.md` — how to route ambiguous intake answers.
- `references/prerequisites.md` — checks + install commands per OS.
- `references/quickstart.md` — hello-world walkthrough, all 4 SDKs.
- `references/tour-of-workflows.md` — 10 stages + deploy, all 4 SDKs.
- `references/tour-of-orchestration.md` — 10 stages, all 4 SDKs.
- `references/ai-agents.md` — AI agent on-ramp, routes to ai-examples and the sibling skill.
