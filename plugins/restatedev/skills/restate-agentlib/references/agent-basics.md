# Agent basics

Lessons:
[01_hello.ts](https://github.com/restatedev/agentlib/blob/main/packages/examples/src/lessons/01_hello.ts),
[04_structured_output.ts](https://github.com/restatedev/agentlib/blob/main/packages/examples/src/lessons/04_structured_output.ts)

## Define once, run in a handler

```ts
import * as restate from "@restatedev/restate-sdk-gen";
import {agent} from "@restatedev/agentlib";
import {z} from "zod";

const assistant = agent({
  model, // see models-and-testing.md
  instructions: "Answer briefly.",
  maxSteps: 6, // model calls per run, including the final answer (default 10)
});

export const hello = restate.service({
  name: "Hello",
  handlers: {
    ask: restate.schemas({input: z.string().min(1), output: z.string()}, function* (message) {
      const result = yield* assistant.run(message);
      if (result.status !== "completed") return result.output ?? result.reason;
      return result.output;
    }),
  },
});
```

A **run** completes one task. A **step** is one model call inside the run,
followed by the tools it asked for. Each request above starts a fresh run
with no history.

## RunResult

```ts
type RunResult<O> = {steps: number; messages: Message[]; usage: RunUsage} & (
  | {status: "completed"; output: O}
  | {status: "stopped"; reason: "max_steps" | "max_tool_calls"; output?: string}
  | {status: "interrupted"; reason: string; output?: string}
);
```

A stopped or interrupted run still gets a short summary in `output`, made by
a tool-free model call (`finalize`, which defaults to the agent's model; pass
`finalize: false` to skip it). When a person reads the answer, that summary
is a fine reply, with `reason` as the fallback. The summary call does not run
`beforeStep`; see [approvals and guardrails](approvals-and-guardrails.md).

```ts
const result = yield* assistant.run(message);
if (result.status !== "completed") return result.output ?? result.reason;
return result.output;
```

When the output feeds the next step of a workflow, a stopped or interrupted
run must not silently become a successful stage. Check the status and throw
a `TerminalError` (from `@restatedev/restate-sdk`):

```ts
const result = yield* triage.run(message);
if (result.status !== "completed") throw new TerminalError(`Triage ${result.status}: ${result.reason}`);
return result.output; // typed by the output schema
```

`usage` holds the run's token totals: `inputTokens`, `outputTokens`,
`totalTokens`, `cachedInputTokens`, and `cacheWriteInputTokens`. A run that
throws carries the same totals on its error; read them with `usageOf` (from
`@restatedev/agentlib`). Record them only for a `TerminalError`, because
Restate retries other errors and the replay counts the same calls again:

```ts
try {
  return yield* triage.run(message);
} catch (error) {
  if (error instanceof TerminalError) console.log("tokens used", usageOf(error)?.totalTokens);
  throw error;
}
```

`models-and-testing.md` lists what the totals include.

## Typed output

Give the agent a Zod `output` schema. The SDK asks the model for that shape,
validates it, and asks again on a mismatch. `result.output` is then typed.

```ts
const classification = z.object({
  category: z.enum(["billing", "technical", "other"]),
  summary: z.string(),
});

const triage = agent({
  model,
  instructions: "Classify the support request and summarize it in one sentence.",
  output: classification,
});

// in a handler with output: classification
const result = yield* triage.run(message);
if (result.status !== "completed") throw new TerminalError(`Triage ${result.status}: ${result.reason}`);
return result.output;
```

Typed output is how agents plug into ordinary code: a router returns
`{specialist}`, a reviewer returns `{approved, feedback}`, and a planner
returns `{questions}`. The schema must be representable as JSON Schema;
transforms are not supported.

## Instructions from context

`instructions` can be a generator that reads the run's `context`, which is
useful when the prompt depends on per-request data:

```ts
const worker = agent<{sources: Source[]}>({
  model,
  *instructions({context}) {
    return `Answer only from these sources: ${context.sources.map((s) => s.id).join(", ")}`;
  },
});
yield* worker.run(question, {context: {sources}});
```

When the agent declares a context type, `run` requires `context`. The same
value reaches every tool and guardrail, but never the model's tool schemas.
This makes it the place for trusted data.

## Skills

A skill is instructions for one kind of task. Give an agent a `SkillSource`;
the model sees each skill's name and description after the instructions and
loads a skill with `load_skill` when a task needs it:

```ts
import {agent, skills} from "@restatedev/agentlib";

const writer = agent({
  model,
  instructions: "You write for a software team.",
  skills: skills([
    {
      name: "release-notes", // lowercase letters, digits and hyphens
      description: "Write release notes from a list of changes.",
      instructions: "Group the changes into Added, Changed and Fixed. Lay them out as in template.md.",
      files: {"template.md": "## <version>\n### Added\n..."},
    },
  ]),
});
```

- A source is `{list, load, read?, retry?}`: plain async functions, each run
  journaled, so they can read files, a repository or a database. `list` runs
  once per run and its result holds for the run, which keeps the cached
  prefix stable. `read` adds `read_skill_file`; the SDK rejects absolute paths
  and `..` before calling it.
- A listed skill may carry a `location` (a folder, a URL at a commit). It is
  journaled with the list and passed back to `load` and `read`, so a source
  over something that changes, such as a branch, pins it per run. The model
  never sees it.
- Skills belong to the run: they survive a handoff. Names must be unique and
  must not clash with run tools. `skills()` throws a `TypeError` for an invalid
  name, an empty description, or a duplicate name. For any other source, the
  run skips such a skill and logs a warning; of two usable skills with the same
  name, it keeps the first, and `load` receives it.
- Lesson 19 shows a source over a folder of `SKILL.md` files. Skill scripts need a shell, which the SDK does not provide.

## Limits

`maxSteps`, `maxToolCalls`, `maxToolDepth`, `maxOutputChars`, and
`maxOutputTokens` bound a run. `maxSteps` is a loop bound, not a retry
count. Using up `maxSteps` (default 10) or `maxToolCalls` (default 128,
counting calls a program makes) stops the run with a summary rather than
failing it. `maxToolDepth` (default 16) bounds nested tool calls,
`maxOutputTokens` (default 4096) each model reply, and `maxOutputChars`
(default 128000) the model's text, a tool's output, the final output, and a
tool's display information and progress. A nested call deeper than
`maxToolDepth` fails with code `depth_limit`, and the tool that made it
receives the error. A reply cut off at `maxOutputTokens`, or model text over
`maxOutputChars`, goes back to the model with a note, and the model tries
again in its next step. A tool's output over the limit fails the call with
code `output_limit`. Display information, progress, or an `afterRun`
replacement over the limit throws a `RangeError`: a mistake in your code,
which Restate retries until you deploy a fix. Retries are configured on the
model adapter and on each tool. For runs that outgrow the model's context, see `toolResults`,
`compaction` and `contextWindow` in `conversations-and-controls.md`.
