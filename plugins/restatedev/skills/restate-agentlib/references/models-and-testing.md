# Models and testing

## Model adapters

Every adapter journals each model call as one step with its own retry
policy. Constructing an adapter needs no credentials, so importing
services stays side-effect free.

Which adapter to use:

| Model                                                                            | Adapter                                                               |
| -------------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| OpenAI, Anthropic, Bedrock, Google, compatible servers, ... | `aiModel` from `@restatedev/agentlib/provider` with an AI SDK provider |
| A model served by another Restate handler                                        | `modelGateway` from `@restatedev/agentlib/provider`                   |
| A client of your own, or a test fixture                                          | `journaledModel` from `@restatedev/agentlib`                          |

### All providers: the AI SDK and `aiModel`

All providers come through the [Vercel AI SDK](https://ai-sdk.dev). The
SDK supplies the provider protocol only; Restate still owns the loop, tools,
retries, and journaling (`aiModel` sets the SDK's own retries to zero). A
throttled call waits as long as the provider asks (`retry-after` headers, or
Gemini's `RetryInfo`) before Restate retries it.

1. Install the provider package, for example `@ai-sdk/openai`, `@ai-sdk/anthropic`,
   `@ai-sdk/amazon-bedrock`, `@ai-sdk/google`, `@ai-sdk/mistral`, or
   `@openrouter/ai-sdk-provider`. The `ai` package itself ships with
   `@restatedev/agentlib`.
2. Create the provider's model object and wrap it with `aiModel`.

```ts
import {anthropic} from "@ai-sdk/anthropic";
import {aiModel} from "@restatedev/agentlib/provider";

export const model = aiModel({model: anthropic("claude-sonnet-5-5"), retry: {maxAttempts: 3}});
```

For OpenAI, install `@ai-sdk/openai` and use:

```ts
import {openai} from "@ai-sdk/openai";
import {aiModel} from "@restatedev/agentlib/provider";

export const model = aiModel({model: openai("gpt-5.6-luna"), retry: {maxAttempts: 3}});
```

The default OpenAI model uses Responses. For a Chat Completions-compatible
server, use `createOpenAI({baseURL, apiKey}).chat(modelId)` instead.

The provider reads its credentials (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, AWS credentials,
`GOOGLE_GENERATIVE_AI_API_KEY`, ...) when the first call is made. Use the
provider's `create*` factory to pass a key, base URL, or custom fetch instead:

```ts
import {createAmazonBedrock} from "@ai-sdk/amazon-bedrock";

const bedrock = createAmazonBedrock({region: "us-east-1"});

export const model = aiModel({model: bedrock("<bedrock model id>")});
```

`aiModel` also accepts `providerOptions` (passed through to the provider), an
OpenTelemetry `tracer`, and `onChunk` for streaming text to a UI.

With a reasoning model, `aiModel` keeps the model's reasoning and sends it back on
later requests; for OpenAI it sets `store: false` unless you set `store`. See
`references/reasoning.md`.


Prompt caching is on by default. `aiModel` marks the instructions (which
follow the tools), the newest message, and the message before the newest
model reply (where the previous request ended) as cache points for Anthropic
and Bedrock, so each step reads the previous step's prefix from the cache; other
providers cache prefixes on their own and ignore these options. Pass
`caching: false` to send none, for example for a Bedrock model without
prompt caching. `aiModel` reports `usage` when the provider gives counts:
`inputTokens` includes cached reads and writes, `cachedInputTokens` counts the
reads, and `cacheWriteInputTokens` the writes.

`result.usage` sums the usage of every model call a run makes: its steps,
in-run compaction, and the summary of a stopped or interrupted run, including
responses the run discards, such as an answer superseded by steering or a reply
cut off at `maxOutputTokens`. It also includes each sub-agent run by
`agentTool` and each fork run by `forkAgentTool`, whether it completes, stops,
or fails. `totalTokens` is `inputTokens + outputTokens`. Calls that
report no usage add nothing. Model calls made by other tools or by hooks,
conversation-store compaction, and calls cancelled before they return are not
counted.

When `run()` throws, `usageOf(error)` returns the totals up to the failure (see
the example in `agent-basics.md`). Record them only for terminal errors: after
a retryable error, Restate retries the invocation, and the replay counts the
same calls again. The totals are attached to the error object in the handler,
so a remote caller of the handler does not receive them. When several runs
throw the same error object, such as two runs in one handler that is
cancelled, `usageOf` returns the totals of the run that threw it last.

`aiModel` passes tool and `output` schemas to AI SDK as they are, so AI SDK's
defaults apply: tool strictness follows provider defaults, and OpenAI
structured output is strict.
OpenAI strict mode requires every property, so use `.nullable()` rather than
`.optional()` in an `output` schema. It also rejects open-ended keys, such as
`z.record()` or `.passthrough()`, and `.nullable()` does not help there. For
those, pass `providerOptions: {openai: {strictJsonSchema: false}}`. Tool input
and output are validated against their Zod schemas either way.

`modelGateway(call)` sends model requests to another Restate handler. Use it
to centralize keys, quotas, or routing in one service:

```ts
import {modelGateway} from "@restatedev/agentlib/provider";

const viaGateway = modelGateway((request) => restate.client(gateway).generate(request));
```

A model is just a generator from `ModelRequest` to `ModelResponse`, so
`journaledModel(async (request, {signal}) => ...)` wraps any other client.

### Retries and failed model calls

A model call makes three attempts by default; pass `retry` to change that. In
`aiModel`, a failure that retrying can't fix, such as a missing API key, a
feature the provider does not support, or a request the provider rejects, fails
at once. Either way, `run()` throws the `TerminalError`, because the next model
call would fail the same way. A custom model that throws a `TerminalError` ends
the run too.

A response that the model can fix goes back to the model instead: a reply cut
off at `maxOutputTokens` or by a content filter, an empty reply, or one that
announces tool calls without making any. The run discards the reply, gives the
model a note that says why, and starts the next step, so a model that keeps
failing ends the run as `stopped`. The journal records the failed call, so a
replay repeats the step without calling the provider. See `error-handling.md`
for how these failures fit the run's other error rules.

## Testing without paid model calls

Give the agents a fixture model and run the services in a real Restate
container with forced replay:

```ts
import {journaledModel, type ModelResponse} from "@restatedev/agentlib";

const fixtureModel = journaledModel(async (request): Promise<ModelResponse> => {
  if (request.tools.length > 0 && request.step === 1)
    return {
      type: "tool_calls",
      calls: [{id: "t1", name: "lookupTicket", input: {ticketId: "T-42"}}],
    };
  if (request.outputSchema)
    return {type: "final", text: "", output: {category: "billing", summary: "…"}};
  return {type: "final", text: "Ticket T-42 is open."};
});
```

- Build agents from a model you can swap: pass it in, read it from a shared
  module, or mock that module in tests.
- Start `RestateTestEnvironment` with
  `container: () => new RestateContainer().alwaysReplay().disableRetries()` (see
  `restate-gen-sdk`'s `running-and-testing.md`), then call the services
  through `@restatedev/restate-sdk-clients`.
- Assert on observable effects: tools that must run once really run once,
  approvals block until resolved, and a race loser is cleaned up.

The repository's
[examples.integration.ts](https://github.com/restatedev/agentlib/blob/main/packages/tests/src/examples.integration.ts)
tests the lessons this way.
