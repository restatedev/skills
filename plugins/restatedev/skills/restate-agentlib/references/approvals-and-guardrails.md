# Approvals and guardrails

Lessons:
[06_human_approval.ts](https://github.com/restatedev/agentlib/blob/main/packages/examples/src/lessons/06_human_approval.ts),
[13_tool_guardrail.ts](https://github.com/restatedev/agentlib/blob/main/packages/examples/src/lessons/13_tool_guardrail.ts),
[08_research_report.ts](https://github.com/restatedev/agentlib/blob/main/packages/examples/src/lessons/08_research_report.ts) (`afterRun`)

## Human approval before a tool runs

Mark the tool `needsApproval: true`, or pass a generator
`(input, context) => boolean` to decide per call. Before running the tool,
the SDK calls `onApproval(request)` to publish the request, then waits on a
durable signal. The wait holds no process and has no default timeout.

```ts
const support = agent({
  model,
  tools: {
    closeTicket: asyncTool({
      description: "Close a ticket after human approval.",
      input: z.object({ticketId: z.string()}),
      needsApproval: true,
      async execute({ticketId}) {
        return await closeTicket(ticketId);
      },
    }),
  },
});

export const ticketApproval = restate.object({
  name: "TicketApproval",
  handlers: {
    *ask(message: string) {
      try {
        const result = yield* support.run(message, {
          *onApproval(request) {
            restate.state().set("approval", request); // or notify Slack/email via restate.run
          },
        });
        if (result.status !== "completed") return result.output ?? result.reason;
        return result.output;
      } finally {
        restate.state().clear("approval"); // never leave a request nobody waits for
      }
    },
    *status() {
      return yield* restate.sharedState().get<ApprovalRequest>("approval");
    },
    *approve({
      invocationId,
      signalName,
      approved,
    }: {
      invocationId: string;
      signalName: string;
      approved: boolean;
    }) {
      restate.invocation(invocationId).signal(signalName).resolve(approved);
      return true;
    },
  },
  options: {handlers: {status: {shared: true}, approve: {shared: true}}},
});
```

An `ApprovalRequest` carries `invocationId`, `signalName`, the proposed
`call` (tool name and arguments), and an optional `question`. If the request
is denied, the tool is skipped and the model gets a denied result.

Variants:

- `onApproval` can also go on the agent definition.
- `onHumanApproval(request, context)` replaces the decision mechanism
  entirely, for example to call an approval service with `restate.client`.
- `humanApprovalTool()` lets the model ask a free-form question and wait for
  a yes or no.
- For a time limit, set `onHumanApproval` and `restate.select` the signal
  against `restate.sleep`.

## Policy the model cannot bypass

Put trusted facts in run `context` and check them in the tool's `before`.
Return a string to deny with that reason, `false` to deny, and nothing or
`true` to allow. A denial goes back to the model, and `execute` never runs.

```ts
const booking = agent<{hotelId: string; quotedTotalCents: number; maxTotalCents: number}>({
  model,
  tools: {
    bookHotel: asyncTool({
      description: "Book the quoted hotel. The application supplies the price.",
      input: z.object({hotelId: z.string()}), // the model cannot set the price
      *before({hotelId}, {context}) {
        if (hotelId !== context.hotelId) return "Only the quoted hotel can be booked";
        if (context.quotedTotalCents > context.maxTotalCents) return "The quote exceeds the budget";
      },
      async execute({hotelId}, {context}) {
        return await reserve(hotelId, context.quotedTotalCents);
      },
    }),
  },
});
yield* booking.run(question, {context: {hotelId, quotedTotalCents, maxTotalCents}});
```

## Agent-level guardrails

A guardrail checks something as it enters the run: the input, a steering
message, a tool call, a tool's output, or the final answer. Every guardrail
returns its decision:

| Return                                             | Means                    |
| -------------------------------------------------- | ------------------------ |
| nothing or `true`                                  | allow                    |
| a string                                           | reject, with that reason |
| `false`                                            | reject, without a reason |
| `{message}`, `{messages}`, `{input}` or `{output}` | replace what is recorded |

`afterTool` is the exception: it returns the output to record. Anything else,
such as an object without the hook's replacement key, is a mistake in the hook
rather than a decision. The hook fails with a `TypeError`, which Restate
retries, so the invocation continues once you deploy a fix. A string always
rejects: to replace the input, return `{message}`, not the new text.

A rejection from `beforeRun`, `beforeTool` or `afterRun` reaches the model,
which decides what to do next, for example answering that it can't help. An
`afterRun` rejection of a stopped or interrupted result is the exception: no
model step follows it, so `run()` throws a `GuardrailError` with code `denied`
and the reason. A `GuardrailError` is a `ToolError` that marks a failure as the
guardrail's rather than the tool's. To stop the run instead, throw `AgentTerminalError` from the hook.

| Hook                            | Checks                                   | A rejection                                                              | Replace with                          |
| ------------------------------- | ---------------------------------------- | ------------------------------------------------------------------------ | ------------------------------------- |
| `beforeRun(messages, ctx)`      | the input, once, before the first step   | reaches the model as a note after the input; the first step goes ahead   | `{message}`                           |
| `beforeSteering(steering, ctx)` | each steering message, before it acts    | drops the message; the run goes on; a `steering-rejected` event says why | `{message}`                           |
| `beforeTool(call, ctx)`         | every tool call, before input validation | goes back to the model as a `denied` result                              | `{input}`, validated like the model's |
| `afterTool(output, ctx)`        | every tool output                        | —                                                                        | returns the output to record          |
| `afterRun(candidate, ctx)`      | the final result                         | gives the model its answer back with a note, for another step            | `{output}`                            |

`beforeStep(request, ctx)` is not a guardrail on data. It runs before each of
the agent's own steps and shapes that one request: a rejection skips that
step's model call and gives the model a note before the next step, and
`{messages}` sends other messages in that request only. A rejection uses up the
step, so a `beforeStep` that rejects every step ends the run as `stopped` with
`max_steps`.

A rejection does not keep data from the model. When the run stops, the summary
of the stopped run sends the recorded history to `finalize` (or the model)
without running `beforeStep`, including the content the hook rejected. To keep
data from every model call, remove it where it enters the history: return
`{message}` from `beforeRun`, or replace a tool's output in `afterTool`. To stop
the run instead, throw `AgentTerminalError` from the hook; `finalize: false`
skips the summary.

### What is recorded

The run's history is what the model sees in later steps, what summaries see,
`result.messages`, and what a `conversation` saves. A run saves its history
when it finishes, completed or stopped; a run that fails saves nothing. The
model also sees the notes about rejections and the answers `afterRun`
rejected, for the rest of the run, but they are never recorded.

| Hook             | Allowed: history keeps  | Rejected: history keeps                         | Replaced: history keeps                                   |
| ---------------- | ----------------------- | ----------------------------------------------- | --------------------------------------------------------- |
| `beforeRun`      | the input               | the input                                       | the replacement, never the original                       |
| `beforeSteering` | the message             | nothing                                         | the replacement, never the original                       |
| `beforeTool`     | the call and its result | the call, and a `denied` result with the reason | the model's own call, and the result of the new input     |
| `afterTool`      | —                       | —                                               | the returned output, never the original                   |
| `afterRun`       | the output              | nothing                                         | the replacement, never the candidate                      |
| `beforeStep`     | —                       | nothing                                         | the original messages: the change is for one request only |

**To keep data from the model and from the saved conversation, remove it where
it enters:** `beforeRun` for the input, `beforeSteering` for steering,
`afterTool` for tool output, `afterRun` for the answer. What these hooks record
is all any later model call sees, including summaries. A rejection does not
remove anything: a rejected input stays in the history. To stop the run before
anything is recorded, throw `AgentTerminalError`.

Two replacements leave the original in the history, so they are not for
redaction:

- `beforeStep`'s `{messages}` changes one request. The summary of a stopped run
  (`finalize`) and compaction summaries do not run `beforeStep` at all. Use it
  to add context to a request, such as retrieved documents.
- `beforeTool`'s `{input}` changes what the tool runs with, not the model's
  call. The model sees its own call next to a result from other arguments and
  cannot tell they changed; for changes it should know about, say so in the
  tool's result.

Restate's journal is separate from the history. It keeps the handler's input and
the result of every journaled step, such as an `asyncTool`'s output before
`afterTool` runs, until Restate's retention removes them. To keep data out of the
journal too, remove it before it enters a journaled step, for example inside the
tool before it returns.

```ts
const support = agent({
  model,
  *beforeRun(messages) {
    const input = messages.at(-1); // the run's input is the last message
    if (input?.role === "user") return {message: redact(input.content)};
  },
  *beforeSteering({message}) {
    if (isOffTopic(message)) return "Steering must be about the current task";
    return {message: redact(message)};
  },
  *afterTool(output) {
    return redactJson(output);
  },
});
```

### Rules for hook code

- Return the decision. A rejection or replacement that is built but not
  returned is ignored, and the hook allows.
- Reject by returning, not by throwing. A thrown error means the check itself
  failed, as described below.
- A hook never edits what it receives; it returns the replacement. What a hook
  receives is typed read-only and is not copied: agentlib
  shares the run's messages with hooks and models, so an edit in place would
  change the history without any check.
- From `beforeStep`, keep the messages you do not change: the SDK validates
  only new ones, then checks that every tool call still has its result.
- Hooks run again when Restate replays the handler, so a decision or
  replacement must come out the same every time: compute it deterministically,
  or inside `run()` so the journal returns the first result.
- `beforeRun`, `beforeSteering` and `afterRun` belong to the root agent. A
  handoff target adds its own `beforeStep`, `beforeTool` and `afterTool`: the
  root's checks run first, and its `afterTool` last.
- For a sub-agent called through `agentTool`, its `beforeRun` checks the input
  the parent model sends.
- A fork started with `forkAgentTool` keeps the run's policies: the root's and the
  active agent's checks apply to its calls, and its approval requests reach the
  run's handlers. It skips the root's `beforeRun` and `afterRun`: its input and
  answer are a tool call of the parent run and its result.

A hook that throws a `TerminalError`, for example because a policy model is
down, doesn't fail the run: the model receives the error, as it does a
rejection. From `beforeTool` or `afterTool`, it is the call's result, with
code `guardrail_failed` and a message that starts with `Policy check failed:`,
or with the code of a `ToolError` the hook threw. From `beforeRun`, `beforeStep`
or `afterRun`, it is a note before the model's next step; from `afterRun` for a
stopped or interrupted result, it fails the run, as a rejection does. From
`beforeSteering`, it rejects the steering message. Throw `AgentTerminalError`
to fail the run instead. Restate retries any other thrown error. See
`error-handling.md` for the full rules.

A hook can also ask a human first: `ctx.approve(call.input, {question,
details})` waits for the decision. `details` is any JSON value, such as the
policy that asked; it reaches `onApproval` and `onHumanApproval` on the
request.

Lesson 08 uses `afterRun` to reject a research finding that cites a source
the worker never read. The tool records reads in `context`, and `afterRun`
compares them with the cited IDs. The worker's model receives the reason and
can read the missing sources before it answers again.
