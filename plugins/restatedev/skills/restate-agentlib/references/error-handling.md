# Error handling

Lessons:
[02_tools.ts](https://github.com/restatedev/agentlib/blob/main/packages/examples/src/lessons/02_tools.ts),
[08_research_report.ts](https://github.com/restatedev/agentlib/blob/main/packages/examples/src/lessons/08_research_report.ts) (`afterRun`),
[11_coding_agent.ts](https://github.com/restatedev/agentlib/blob/main/packages/examples/src/lessons/11_coding_agent.ts)

The type of an error decides its outcome: the model handles it, the run stops,
or Restate retries it. The same rule applies in a tool, an `asyncTool`, a
handler a tool calls, a sub-agent, a guardrail, and a hook. Where the error is
thrown only decides how the model receives it, and in the few places where no
model step can follow, a `TerminalError` fails the run instead. This page
explains the three outcomes, what the model receives, and how errors behave
across journaled steps and RPC calls.

## Choose the error to throw

| Error                                  | Who retries | What happens inside a run                                                                                                        |
| -------------------------------------- | ----------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `TerminalError`, including `ToolError` | The model   | The nearest agent gives the error to its model: as the tool call's result, or as a note before the model's next step.            |
| `AgentTerminalError`                   | Nobody      | The run fails, and so does every agent run that encloses it. The handler fails too, unless it catches the error.                 |
| Any other error                        | Restate     | Restate retries the step or the invocation, and by default pauses the invocation when its retries run out, so you can resume it. |

- Throw `TerminalError` (from `@restatedev/restate-sdk`) when the model should
  handle a failure, for example by trying another approach.
- Throw `ToolError` (from `@restatedev/agentlib`) to also give the model an
  error code, such as `not_found`. A `ToolError` is a `TerminalError`.
- Throw `GuardrailError` (from `@restatedev/agentlib`) from a guardrail to mark
  the failure as the guardrail's rather than the tool's. It is a `ToolError`
  with code `guardrail_failed` by default.
- Throw `AgentTerminalError` (from `@restatedev/agentlib`) when the agent must
  stop immediately, for example because the request it serves is no longer
  valid.
- Let every other error propagate. Restate retries it, which is right for
  transient failures, and for mistakes in your code that a deploy can fix.

No setting changes what an error means. Outside an agent run, a
`TerminalError` or `ToolError` is an ordinary terminal error. Restate's
`CancelledError` is a `TerminalError` too, but it means the invocation was
cancelled, so it never reaches the model: see [Cancellation](#cancellation).

```ts
import {TerminalError} from "@restatedev/restate-sdk";
import {AgentTerminalError, asyncTool, ToolError} from "@restatedev/agentlib";

// In an agent's tools, with the customer in the run's context.
refund: asyncTool({
  description: "Refund an order.",
  input: z.object({orderId: z.string()}),
  async execute({orderId}, {context, callId}) {
    const order = await orders.get(orderId); // a network error: retried
    if (!order) throw new ToolError(`No order ${orderId}`, "not_found"); // the model gets the code
    if (order.refunded) throw new TerminalError("The order was already refunded"); // the model decides
    if (order.customerId !== context.customerId)
      throw new AgentTerminalError("The order belongs to another customer"); // the run fails
    return await payments.refund(orderId, {idempotencyKey: callId});
  },
}),
```

## What the model receives

### From a tool call

A `TerminalError` from any part of a tool call fails that call, not the run.
That includes input parsing, `beforeTool`, the tool's `before`,
`needsApproval`, the approval handlers, `execute`, the tool's `after`,
`afterTool`, the `onProgress` handler for the call's start, display and
progress events, and anything the tool calls: another tool, a Restate handler,
an MCP server, or a sub-agent. The call's result is:

```json
{"status": "error", "code": "not_found", "message": "No order 42"}
```

The code is the `ToolError`'s code, or `tool_failed` for any other
`TerminalError`. The other calls in the same batch keep running, and the model
sees every result in its next step, so a failed call costs no extra step.

The agent's `beforeTool` and `afterTool` policies are guardrails, so the model
can tell their failures from the tool's: a `TerminalError` without a code from
either one becomes a `GuardrailError` with code `guardrail_failed`, and its
message starts with `Policy check failed:`. A `ToolError` thrown by a policy
keeps its code.

A rejection by `beforeTool`, the tool's `before`, or a human approver is
recorded with status `denied` instead:
`{"status": "denied", "message": "Searching is off today"}`.

`context.callTool()` throws a `ToolError` when the nested call fails, with the
nested result's code. Catch it to recover inside the tool, or let it fail the
outer call with the same code.

### From anywhere else in the run

A `TerminalError` from the rest of the run reaches the model as a note: a
user message from the runtime, before the model's next call.

```json
{"source": "agent-runtime", "type": "error", "origin": "beforeStep", "code": "pii_detected", "message": "Remove the account number."}
```

`origin` says where the error came from. `code` appears when the error is a
`ToolError` or the SDK provides one. Notes are provisional: the model sees them
for the rest of the run, but they are never saved to the conversation or
returned in `result.messages`. Each note emits an `error-note` progress event
with the same `origin`, `code` and `message`, so `onProgress` can record what
went wrong:

```ts
const result = yield* support.run(message, {
  *onProgress(event) {
    if (event.type === "error-note") yield* recordProblem(event.origin, event.code, event.message);
  },
});
```

| Where the `TerminalError` is thrown                                 | What the run does                                                         | Uses up the step |
| ------------------------------------------------------------------- | ------------------------------------------------------------------------- | ---------------- |
| `beforeRun`                                                         | Adds a note after the input. The first model call goes ahead.             | No               |
| The `instructions` function                                         | Skips the step's model call and adds a note.                              | Yes              |
| `beforeStep`                                                        | Skips the step's model call and adds a note.                              | Yes              |
| The model's response, when the model can fix it (see below)         | Discards the response and adds a note.                                    | Yes              |
| `afterRun`, for a completed answer                                  | Gives the model its answer back with a note, so it can revise the answer. | Yes              |
| `afterRun`, for a stopped or interrupted result                     | Fails the run: no model step follows.                                     | —                |
| `beforeSteering`                                                    | Rejects the steering message; the run continues.                          | —                |
| The model call itself                                               | Fails the run: another call would fail the same way.                      | —                |
| Loading or saving the conversation, or listing skills               | Fails the run: no model step can follow.                                  | —                |
| The `onProgress` handler, for run events and a tool call's end      | Fails the run.                                                            | —                |

Every step that ends in a note counts toward `maxSteps`. A hook that fails on
every step ends the run as `stopped` with reason `max_steps`, with the usual
summary.

When a run fails with a `TerminalError` and that run is a sub-agent called
through `agentTool` or a fork started by `forkAgentTool`, the parent's model
receives the error as the call's result. A sub-agent or fork that stops early
becomes a result with code `agent_stopped`, and an interrupted one a cancelled
call.

### Codes the SDK uses

| Code                  | Meaning                                                                                       |
| --------------------- | --------------------------------------------------------------------------------------------- |
| `tool_failed`         | A tool call failed with a `TerminalError` that has no code.                                   |
| `guardrail_failed`    | `beforeTool` or `afterTool` failed with a `TerminalError` that has no code.                   |
| `tool_error`          | The default code of a `ToolError`, for example an MCP result with `isError`.                  |
| `invalid_input`       | The model's arguments failed the tool's input schema.                                         |
| `invalid_output`      | The tool returned something that isn't JSON.                                                  |
| `output_limit`        | A tool's output, or the model's text, is longer than `maxOutputChars`.                        |
| `unknown_tool`        | The model called a tool the agent doesn't have.                                               |
| `depth_limit`         | A nested call went deeper than `maxToolDepth`. Only that call fails.                          |
| `agent_stopped`       | A sub-agent or fork stopped before finishing; the message carries its summary.                |
| `denied`              | `beforeRun`, `beforeStep` or `afterRun` rejected the input, a step, or an answer.             |
| `invalid_response`    | The model's response was empty, announced tool calls without any, or had an invalid batch.    |
| `incomplete_response` | The model's response was cut off at `maxOutputTokens` or stopped by a content filter.         |

Some built-in tools add their own codes, such as `program_error` for a failed
code-mode program, `unknown_ref` for a tool result reference that doesn't
exist, and `fork_limit` and `fork_nested` from `forkAgentTool` (see
`multi-agent.md`).

## Stop the run with `AgentTerminalError`

An `AgentTerminalError` never reaches a model. Thrown from any place, it fails
the run:

- The other calls running in the same batch are interrupted.
- Nothing is saved to the conversation, as for any run that throws.
- Every agent run that encloses this one fails too: a parent that called this
  agent through `agentTool`, the run a fork was started from, and the agent
  whose tool called the handler that threw it.
- The handler fails with the error, unless it catches it.

```ts
import {isAgentTerminal} from "@restatedev/agentlib";

// A handler that turns a refusal into an answer instead of a failed invocation.
*handle(message: string) {
  try {
    const result = yield* support.run(message);
    return result.status === "completed" ? result.output : result.reason;
  } catch (error) {
    if (isAgentTerminal(error)) return `Request refused: ${(error as Error).message}`;
    throw error;
  }
},
```

Use `isAgentTerminal(error)` rather than `instanceof` when the error may have
crossed a journaled step or an RPC call, as described below. Pass
`{errorCode}` to set the status the handler's caller receives; the default is
500. Any code works, including 413: the run never treats an
`AgentTerminalError` as a context overflow. `RunFailedError` is a deprecated
alias of `AgentTerminalError`.

## Let Restate retry everything else

A plain `Error` means the failure may go away on its own, or that the code has a
mistake that a deploy can fix. Restate retries it:

- In an `asyncTool`, under the tool's `retry` policy (default: three attempts).
  When the attempts run out, the step fails with a `TerminalError`, and the
  model receives `tool_failed`.
- In `restate.run`, under the policy you pass.
- Anywhere else, Restate retries the whole invocation, which replays the
  journal and continues where the run was. When the invocation's retries run
  out, Restate pauses it by default, so you can fix the cause and resume it.

The SDK throws a plain error for mistakes in your code, so that a fix can resume
the invocation instead of failing it:

- A hook returns something that isn't a decision, such as an object without
  the hook's replacement key, or `beforeTool` returns input that isn't JSON
  (`TypeError`).
- `beforeStep` returns messages that are invalid or break the order of tool
  calls and results, or `beforeRun` or `beforeSteering` returns an empty
  message (`TypeError`).
- `afterRun` replaces the output with a value that fails the output schema,
  isn't text for an agent without an output schema, isn't JSON, or is longer
  than `maxOutputChars` (`TypeError`, `RangeError`).
- A tool's display information or progress is longer than `maxOutputChars`
  (`RangeError`). Events a sub-agent or fork forwards are not checked: the
  sub-agent's own limits apply to them.
- `skills()` receives a skill with an invalid name, an empty description, or
  the name of another skill (`TypeError`). It throws when it's called, usually
  when the service starts.

Invalid input from the caller, such as an empty input or invalid stored
history, fails the run with a `RunInputError`: no deploy can fix it.

## Model call failures

A response the model can fix goes back to the model. The run discards the
response, adds a note with the reason, and starts the next step:

- The reply was cut off at `maxOutputTokens` or stopped by a content filter
  (`incomplete_response`).
- The reply was empty, or announced tool calls without making any
  (`invalid_response`).
- The batch had two handoffs or reused a tool-call ID. No call in it starts
  (`invalid_response`).
- The text is longer than `maxOutputChars` (`output_limit`).

The journal records a failed model call, so a replay repeats the step without
calling the provider. The tokens the discarded response used still count in
`result.usage`.

A model call that fails is retried, three attempts by default; pass `retry` to
the model to change that. In `aiModel`, a failure that retrying can't fix, such
as a missing API key, a feature the provider doesn't support, or a request the
provider rejects, fails at once. Either way, `run()` throws the
`TerminalError`, because the next model call would fail the same way. A custom
model that throws a `TerminalError` ends the run too.

When the model rejects a request as longer than its context window, the run
shrinks its view of the history once and asks again. If that request overflows
too, the run fails with an error that `isContextOverflow(error)` recognizes. See
`conversations-and-controls.md`.

## Guardrail rejections

A guardrail rejects by returning a reason or `false`:

- A rejection from `beforeRun`, `beforeStep` or `afterRun` reaches the model as
  a note with code `denied`, in the same place as a `TerminalError` from that
  hook.
- A rejection from `beforeTool` or a tool's `before` becomes the call's result,
  with status `denied`.
- A rejection from `beforeSteering` drops the steering message.

The SDK represents a rejection from `beforeRun`, `beforeStep` or `afterRun` as a
`GuardrailError` with code `denied`. A `GuardrailError` is a `ToolError`, so it
behaves like one; its class and code tell the model and your code that a
guardrail failed, not the tool or step it guards. When `afterRun` rejects a
stopped or interrupted result, no model step can follow, so `run()` throws that
`GuardrailError`.

Throw from a hook only when the check itself failed, for example because a
policy service is down; the model then receives the failure in the same place.
Throw `GuardrailError` to mark the failure as the guardrail's anywhere, for
example from a tool's own `before`. To stop the run on a rejection, throw
`AgentTerminalError` from the hook.

A `beforeStep` rejection does not keep content from the model: when the run
stops, the summary of the stopped run sends the recorded history to `finalize`
without running `beforeStep`. Set `finalize: false`, throw
`AgentTerminalError`, or remove the content where it enters the history.

A rejected input stays in the run's history and is saved when the run
completes. Return `{message}` from `beforeRun` to record a redacted input
instead, or throw `AgentTerminalError` to stop before anything is recorded. See
`approvals-and-guardrails.md` for each hook.

## Errors across journaled steps and RPC

A journaled step (`asyncTool`, `restate.run`, a model call) and an RPC call to
another handler return a failure as a plain `TerminalError` with its message,
status code and metadata. The error's class is lost. The SDK stores what it
needs in the metadata, so the meaning survives:

- A `ToolError` keeps its code. A handler called through `handlerTool` or
  `serviceTool` can throw `new ToolError("Ticket not found", "not_found")`, and
  the agent's model receives that code.
- An `AgentTerminalError` stays recognizable with `isAgentTerminal(error)`, so
  a handler that throws one fails the agent run that called it, also through an
  intermediate handler, and also on replay.

Read a local `ToolError`'s code from `error.toolCode`. `error.code` is the
numeric status that every `TerminalError` has. Error metadata requires Restate
1.6 or later.

## Cancellation

Cancellation is never sent to the model as something to recover from:

- **The invocation is cancelled.** The Restate SDK throws a `CancelledError` at
  the run's next `yield*`, wherever the run is: in a tool, a hook, or the model
  call. The run lets it through, without a note or a tool result, and skips its
  fallbacks. The other running calls are interrupted, child invocations the
  run started are cancelled, nothing is saved, and the handler fails with the
  `CancelledError`.
- **The run is interrupted** through `interruptSignal` or `controls.interrupt`.
  This isn't an error: the run ends with status `interrupted`. Calls that
  didn't finish are recorded with `{status: "cancelled"}`. The model that
  writes the run's summary sees those results, and its request tells it not to
  claim that cancelled work succeeded.
- **The model cancels a background call** with `cancelOperationTool()`. The
  call's result is `{status: "cancelled"}`, which the model sees because it
  asked for the cancellation.

A child invocation that something else cancels is different. Restate reports
its cancellation to the caller as a failed call, and the SDK rebuilds every
failed call as a plain `TerminalError`, not a `CancelledError`. The tool call
that awaited the child fails, and the model receives `tool_failed`.

## Run endings that aren't errors

Reaching `maxSteps` or `maxToolCalls` stops the run with status `stopped`, and
an interrupt ends it with status `interrupted`. Both are results, not errors:
check `result.status` before you use `result.output`.

Some failures have a fallback instead of an outcome:

- If a summary for in-run compaction fails, the run truncates older messages.
- If conversation compaction fails when a run starts, the run continues with
  the history it loaded and logs a warning.
- If the summary of a stopped run fails, or is longer than `maxOutputChars`,
  the result gets a fixed text instead, or no summary when even that text is
  too long.
- A skill source other than `skills()` that lists a skill with an invalid
  name, an empty description, or a name already listed: the run skips that
  skill and logs a warning. Of two usable skills with the same name, the run
  keeps the first, and `load` receives it.

An `AgentTerminalError` and cancellation skip these fallbacks and fail the run.

## What a failed run keeps

A run that throws saves nothing to its conversation, so the next run starts
from the history before the failed one. It does not return partial messages
either. `usageOf(error)` returns the token totals up to the failure; record
them only for terminal errors, because Restate retries other errors and the
replay counts the same calls again.
