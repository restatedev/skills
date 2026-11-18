---
name: restate-agentlib
description: >
  Build durable AI agents with the Restate agentlib (@restatedev/agentlib):
  an agent runs inside a Restate generator handler, and every model call, tool
  result, approval, and sub-agent step is journaled, so work survives crashes,
  deploys, and long waits. Use when the user wants to build, extend, or debug
  an AI agent on Restate; mentions @restatedev/agentlib, agent(), asyncTool,
  handoff, agentTool, forkAgentTool, serviceTool, mcpTools, needsApproval, steering, or
  interrupting an agent; or wants multi-agent workflows (routing, racing,
  orchestrator-workers, evaluator-optimizer, peer review) composed from Restate
  services. Also use when teaching someone the SDK step by step.
---

# Restate agents

An agent is a value you define once and `run` from any Restate handler. The
handler owns everything around the run: where history is stored, who
approves, what happens after. The agent owns the model/tool loop. Because the
run happens inside a Restate invocation, completed model calls and tool
results replay from the journal instead of repeating.

```ts
import * as restate from "@restatedev/restate-sdk-gen";
import {agent} from "@restatedev/agentlib";
import {openai} from "@ai-sdk/openai";
import {aiModel} from "@restatedev/agentlib/provider";

const assistant = agent({model: aiModel({model: openai("gpt-5.6-luna")})});

export const hello = restate.service({
  name: "Hello",
  handlers: {
    *ask(question: string) {
      const result = yield* assistant.run(question);
      return result.status === "completed" ? result.output : result.reason;
    },
  },
});
```

Agents build on the generator SDK. Load the `restate-gen-sdk` skill as well:
its rules (generators, `yield*`, no `Promise.all`, journal every side effect)
apply to agent handlers and tools too.

## Setup

- Dependencies: `@restatedev/agentlib`, `@restatedev/restate-sdk`,
  `@restatedev/restate-sdk-gen`, and `zod`. The agentlib lives in
  https://github.com/restatedev/agentlib.
- A model adapter. All providers, including OpenAI, go through the
  [Vercel AI SDK](https://ai-sdk.dev): install the provider package (for example
  `@ai-sdk/openai`) and wrap its model with `aiModel` from
  `@restatedev/agentlib/provider`. See `references/models-and-testing.md`, and
  `references/reasoning.md` for reasoning models.
- Serve the handlers with `serve({services, port: 9080})` from
  `@restatedev/restate-sdk` and register them, as in the `restate-gen-sdk`
  skill.

## Learning path

The numbered lessons in the repository introduce one concept at a time:
https://github.com/restatedev/agentlib/tree/main/packages/examples/src/lessons.
Follow the same order when teaching or when growing an agent step by step.

| #   | Concept                              | API                                                          | Reference                                  |
| --- | ------------------------------------ | ------------------------------------------------------------ | ------------------------------------------ |
| 01  | One agent in one handler             | `agent({model})`, `run`, `RunResult`                         | `references/agent-basics.md`               |
| 02  | A tool with validated arguments      | `asyncTool`, `tool`                                          | `references/tools.md`                      |
| 03  | Conversation history in an object    | `run(input, {messages})`                                     | `references/conversations-and-controls.md` |
| 04  | Typed output                         | `agent({output})`                                            | `references/agent-basics.md`               |
| 05  | A bounded draft-and-review loop      | two agents, distinct run `name`s                             | `references/multi-agent.md`                |
| 06  | Human approval before a tool         | `needsApproval`, `onApproval`                                | `references/approvals-and-guardrails.md`   |
| 07  | Race two agents                      | `restate.spawn`, `restate.any`                               | `references/multi-agent.md`                |
| 08  | Parallel workers over RPC            | `restate.all` of `restate.client` calls                      | `references/multi-agent.md`                |
| 09  | Steer and interrupt a running turn   | `LazyStateConversation`, `steeringSignal`, `interruptSignal` | `references/conversations-and-controls.md` |
| 10  | A Restate service as a tool          | `serviceTool`                                                | `references/tools.md`                      |
| 11  | Tools backed by a sandbox            | run `context`, `asyncTool`                                   | `references/tools.md`                      |
| 12  | Route to a specialist                | structured-output router                                     | `references/multi-agent.md`                |
| 13  | Enforce a policy before an action    | tool `before`, agent guardrails                              | `references/approvals-and-guardrails.md`   |
| 14  | Web search with a custom tool       | example-local `asyncTool` calling Tavily                       | `references/tools.md`                      |
| 15  | Agents reviewing each other over RPC | typed services per role                                      | `references/multi-agent.md`                |
| 16  | Stream progress to subscribers       | `onProgress`, `context.progress`                             | `references/conversations-and-controls.md` |
| 17  | Tools from an MCP server             | `mcpTools` (`/mcp`)                                          | `references/tools.md`                      |
| 18  | Serve an agent to AG-UI clients      | `onProgress`, `onApproval`, client-run tools as AG-UI events | `references/conversations-and-controls.md` |
| 19  | Skills from a folder                 | `agent({skills})`, a `SkillSource` over files                | `references/agent-basics.md`               |

**When teaching:** go one row at a time. Show the smallest code that adds
the new concept to the previous step, run it with curl, and explain what
Restate journals. Move on only when the learner is ready.

## Find the right tool for a goal

| Goal                                                           | Use                                                                                       |
| -------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| Call an API from a tool                                        | `asyncTool` with `retry`                                                                  |
| Compose Restate operations inside a tool (RPC, sleep, signals) | `tool` with `*execute`                                                                    |
| Let the model call one or all handlers of a Restate service    | `handlerTool` / `serviceTool`                                                             |
| Remote tools                                                   | `mcpTools`; your own registered services: `restateTools` (`/discovery`) |
| Many tools, load schemas on demand                             | `deferred: true` (tool search is on by default)                                           |
| Let the model script several tool calls in one step            | Code mode (on by default); `programTool()` (`/program`) to configure it                   |
| Keep working while a slow tool runs                            | `background: true`, `sleepTool()`, `cancelOperationTool()`                                |
| A human decides first                                          | `needsApproval`, or `humanApprovalTool()` for free-form questions                         |
| Enforce policy the model cannot bypass                         | tool `before`, agent `beforeTool` / `afterRun`                                            |
| Keep personal data from the model                              | `beforeRun`, `beforeSteering`, `afterTool`, `afterRun`: they set what is recorded         |
| Check or reject steering messages                              | `beforeSteering`                                                                          |
| Delegate a subtask, keep control                               | `agentTool(subAgent, {description})`                                                      |
| Split work across copies that share the conversation           | `forkAgentTool({description})`                                                                 |
| Hand over the conversation                                     | `handoff(specialist)`                                                                     |
| Independent, separately retried agents                         | one Restate service per agent, called with `restate.client`                               |
| A long run outgrows the model's context                        | `toolResults` (on by default), `compaction`, `contextWindow`                              |
| Tools that depend on the request, such as a UI's own tools     | `run(input, {tools})`                                                                     |
| Many task-specific instructions, loaded only when needed       | `agent({skills})` with a `SkillSource`, or `skills([...])` for skills in code             |
| Decide what a failure does: model retries, run stops, or Restate retries | `TerminalError` / `ToolError`, `AgentTerminalError`, a plain error; see `references/error-handling.md` |

## Rules to check before finishing

- [ ] Define agents at module scope. Run them inside handlers or tools with
      `yield* agent.run(...)`.
- [ ] Handle every `RunResult` status: `completed`, `stopped` (hit
      `maxSteps` or `maxToolCalls`), and `interrupted`. Do not read `output` as the answer
      unless the status is `completed`.
- [ ] Give each run a distinct `name` when one handler runs the same agent
      more than once (loops, races), so journal entries and approval signals
      stay apart.
- [ ] Tool side effects run in `asyncTool.execute` or `restate.run`. Code in a
      `tool` generator outside those is replayed.
- [ ] Give external writes an idempotency key (`context.callId` is stable
      across retries).
- [ ] Throw `TerminalError` for failures the model should handle, or
      `ToolError` to also give the model an error code. From a tool or a hook,
      the model receives the error and the run continues. Throw
      `AgentTerminalError` to stop the run. Let other errors propagate:
      Restate retries them. See `references/error-handling.md`.
- [ ] Trusted values (prices, user IDs, limits) come from run `context`,
      never from model-supplied tool arguments.
- [ ] Approval requests and active-turn IDs stored in state are cleared in a
      `finally`.
- [ ] Tests use a fixture model (`journaledModel`) against
      `RestateContainer().alwaysReplay().disableRetries()`. No paid model calls.
