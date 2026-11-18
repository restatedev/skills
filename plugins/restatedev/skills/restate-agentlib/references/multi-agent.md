# Multiple agents

Lessons:
[05_evaluator_optimizer.ts](https://github.com/restatedev/agentlib/blob/main/packages/examples/src/lessons/05_evaluator_optimizer.ts),
[07_racing_agents.ts](https://github.com/restatedev/agentlib/blob/main/packages/examples/src/lessons/07_racing_agents.ts),
[08_research_report.ts](https://github.com/restatedev/agentlib/blob/main/packages/examples/src/lessons/08_research_report.ts),
[12_routing.ts](https://github.com/restatedev/agentlib/blob/main/packages/examples/src/lessons/12_routing.ts),
[15_peer_review_rpc.ts](https://github.com/restatedev/agentlib/blob/main/packages/examples/src/lessons/15_peer_review_rpc.ts),
[20_forked_agents.ts](https://github.com/restatedev/agentlib/blob/main/packages/examples/src/lessons/20_forked_agents.ts)

Multi-agent systems are ordinary Restate composition. The handler decides
the control flow in plain code. Agents are steps that return typed output.

## Choose the boundary

| Shape                                                      | Use                                             | Each agent is                                                  |
| ---------------------------------------------------------- | ----------------------------------------------- | -------------------------------------------------------------- |
| Code decides the flow                                      | Run several agents in one handler               | a step in one invocation                                       |
| The model decides to delegate a subtask                    | `agentTool(subAgent, {description})`            | a tool call; fresh history; result returned to the parent      |
| The model splits work across copies of itself              | `forkAgentTool({description})`                       | a tool call; the history so far; result returned to the parent |
| The model hands over the conversation                      | `handoff(specialist)`                           | the same run, continued by another agent with the full history |
| Independent, retried, observable, or owned by another team | One Restate service per agent, `restate.client` | a separate durable invocation                                  |

## Route, then answer

```ts
const route = agent({
  model,
  instructions: "Choose the one specialist best suited to the question. Classify only.",
  output: z.object({specialist: z.enum(["billing", "account", "product"])}),
});
const specialists = {billing: agent({...}), account: agent({...}), product: agent({...})};

function* (question) {
  const routed = yield* route.run(question);
  if (routed.status !== "completed") throw new TerminalError(`Router ${routed.status}`);
  const result = yield* specialists[routed.output.specialist].run(question);
  if (result.status !== "completed") return result.output ?? result.reason;
  return result.output;
}
```

## Draft and review, bounded

Loop over two structured agents. Give each run a distinct `name` so their
journal entries and approval signals stay apart.

```ts
for (let i = 1; i <= 3; i++) {
  const draft = yield* writer.run(JSON.stringify({task, previousCode: code, feedback}), {
    name: `write-${i}`,
  });
  if (draft.status !== "completed") throw new TerminalError(`Writer ${draft.status}`);
  code = draft.output.code;
  const review = yield* reviewer.run(JSON.stringify({task, code}), {name: `review-${i}`});
  if (review.status !== "completed") throw new TerminalError(`Reviewer ${review.status}`);
  if (review.output.approved) return code;
  feedback = review.output.feedback;
}
```

## Race, keep the first success

```ts
function* answerWith(name: string, candidate: Agent<undefined>, question: string) {
  const result = yield* candidate.run(question, {name});
  if (result.status !== "completed") throw new TerminalError(`${name} ${result.status}`);
  return result.output;
}

const tasks = Object.entries(candidates).map(([name, c]) =>
  restate.spawn(answerWith(name, c, question)),
);
try {
  return yield* restate.any(tasks); // a fast failure does not win
} finally {
  for (const task of tasks) task.interrupt(); // cancel the loser's model call and tools
  yield* restate.allSettled(tasks); // join so cleanup runs
}
```

## Orchestrator and workers over RPC

Make the worker a service. The orchestrator fans out typed calls and
synthesizes the results. Each worker has its own journal, retries, and entry
in the UI. Cancelling the orchestrator cancels the unfinished workers.

```ts
export const researchWorker = restate.service({
  name: "ResearchWorker",
  handlers: {
    *research({question, sources}: WorkerInput) {
      const result = yield* researcher.run(question, {
        context: {sources, readSourceIds: new Set()},
      });
      if (result.status !== "completed") throw new TerminalError(`Research ${result.status}`);
      return result.output;
    },
  },
});

// in ResearchReport.generate:
const plan = yield* planner.run(topic);
if (plan.status !== "completed") throw new TerminalError(`Planner ${plan.status}`);
const findings = yield* restate.all(
  plan.output.questions.map((question) =>
    restate.client(researchWorker).research({question, sources}),
  ),
);
const report = yield* writer.run(JSON.stringify({topic, findings}));
if (report.status !== "completed") throw new TerminalError(`Writer ${report.status}`);
return report.output;
```

## Agents exchanging messages

Lesson 15 has an author service and a reviewer service, plus a lead that
passes drafts and feedback between them as typed request and response
bodies. No mailbox or agent-to-agent protocol is needed: durable RPC is the
channel, and a replay never repeats a completed exchange.

## Sub-agents and handoffs inside one run

```ts
const lead = agent({
  model,
  tools: {
    research: agentTool(researcher, {description: "Research one question in depth."}),
    toBilling: handoff(billing, "Transfer billing questions to the billing specialist."),
  },
});
```

- **`agentTool`**: the sub-agent gets the task as a fresh conversation plus
  the parent's `context`. Its answer becomes the tool result. Its approvals
  and progress flow through the parent, and its token usage counts toward
  the parent's `result.usage`, also when it fails. A sub-agent that stops early
  becomes a tool error with code `agent_stopped` carrying its summary; an
  interrupted one, a cancelled call. A sub-agent run that fails with a
  `TerminalError` becomes a tool error with code `tool_failed`, or the code of
  a `ToolError`. An `AgentTerminalError` in the sub-agent fails the parent run
  too. Pass `background: true` to let the parent keep working meanwhile.
- **`handoff`**: after the current tool batch settles, the target agent's
  instructions and tools take over the same run and history. Run-level
  tools stay available.

## Forks: copies that share the conversation

`forkAgentTool` starts a copy of the running agent to do one task. A sub-agent from
`agentTool` starts with only its task; a fork starts from the conversation up
to the step that called the tool, so it builds on what the run already found.

```ts
const investigator = agent({
  model,
  instructions: "Investigate incidents. Check plausible causes in parallel with investigate.",
  tools: {
    readLogs,
    investigate: forkAgentTool({
      description: "Check one hypothesis with a copy of yourself that sees this conversation.",
    }),
  },
});
```

- **What a fork copies**: the run as it stood at the step of the call: its
  history, root and active agent, instructions, policies, limits, run-level
  tools, skills, and the deferred tools search loaded. Its first request
  repeats that step's request plus the task, so the provider serves the
  shared part from its prompt cache. Results the run saw as previews, or that
  compaction folded into a note, stay readable with `read_tool_result`.
- **Fork or sub-agent**: fork when the subtask needs what the run already
  learned and the same model and tools. Use `agentTool` for a specialist with
  its own model, instructions, or tools, or for work that needs a clean context.
- **Only the reply returns**: the fork's answer becomes the tool result. It
  replies in plain text, also when the agent has an `output` schema: the schema
  describes the run's final answer, and the fork's reply is a report for the
  parent. Its own tool calls stay out of the parent's history, and it saves
  nothing to the conversation. Its token usage counts toward the parent's `result.usage`, its
  approval requests reach the run's handlers, and interrupting the parent
  interrupts it. `background: true` and `needsApproval` work as on any tool.
- **One level**: a fork cannot start forks or sub-agents. `forkAgentTool` and
  `agentTool` called from a fork return an error with code `fork_nested`.
- **Concurrency**: `maxConcurrent` (default 6) bounds how many forks one fork
  tool runs at once in a run. Each fork tool counts only its own forks. A call
  past the limit returns an error with code `fork_limit`; the model can fork
  again once one finishes.
- **Hooks**: a fork skips the root's `beforeRun` and `afterRun`, because its
  input and answer are a tool call of the parent and its result. `beforeStep`,
  `beforeTool`, and `afterTool` apply as in the parent. `beforeStep` and an
  `instructions` function run again for the fork and see its own step numbers
  and run name: keep them deterministic so the fork's request matches the cache.
- **Limits to know**: a fork sees the parent's pending background calls but
  cannot cancel them or receive their results. It runs inside the parent's
  invocation, so its steps add to the parent's journal; for long, independent
  work, use one service per agent instead.
