# Reasoning

Reasoning models think before they answer or call a tool. `aiModel` keeps that
reasoning and sends it back on later requests, so the model builds on its own
thinking instead of redoing it: across the steps of a tool loop, and in later
turns of a conversation.

## Turn it on

Reasoning is the provider's feature, so you enable it through `providerOptions`.
OpenAI's reasoning models and Gemini 3 reason by default; Claude needs thinking
turned on.

```ts
import {createAnthropic} from "@ai-sdk/anthropic";
import {aiModel} from "@restatedev/agentlib/provider";

export const model = aiModel({
  model: createAnthropic()("claude-sonnet-5-5"),
  providerOptions: {anthropic: {thinking: {type: "adaptive"}}},
});
```

For OpenAI, set `reasoningEffort`; for Gemini, `thinkingConfig`.

## What is kept

Each model response keeps its reasoning on the assistant message's `reasoning`
field, exactly as the provider returned it:

| Provider  | What it returns                                                        |
| --------- | ---------------------------------------------------------------------- |
| Anthropic | Thinking blocks: a readable summary and a signature with the full thinking |
| OpenAI    | Reasoning items, encrypted, with a summary if you ask for one           |
| Gemini 3  | Thought signatures on tool calls and normal text, and thought text if you ask for it |

Tool steps and final answers both keep it. It is saved with the conversation and
comes back on replay and in later turns. Text signatures are stored alongside their
original text pieces, which are sent back unchanged even when structured output
is reformatted in the combined answer.

Reasoning is not always encrypted: it can include readable text. When an output
policy changes an answer, that answer's reasoning and signatures are dropped so
they cannot carry redacted details into saved history or the next request. A policy
that leaves the answer unchanged preserves them. This does not redact reasoning
from earlier tool steps or text already delivered through streaming callbacks.

Later requests pass the saved reasoning through the selected model's AI SDK
adapter. Whether another model accepts or uses it depends on that provider.

Don't edit it: providers reject reasoning that was changed.

## OpenAI storage

`aiModel` defaults to `store: false` for both OpenAI Chat Completions and Responses;
you can override it through `providerOptions.openai.store`. Both API paths remain
supported. 

You can set `providerOptions: {openai: {store: true}}`. The reasoning then goes back
as references to OpenAI's stored copies, and OpenAI deletes those after about 30
days, or when you delete the response. A conversation that refers to a deleted copy
fails on every later turn with `Item with id 'rs_…' not found`. So does a
conversation started with `store: false` once you switch to `store: true`. Keep the
default for conversations that live long.

## Tokens and summaries

Reasoning sent back counts as input tokens on later requests. `aiModel` records how
many tokens each response's reasoning took (Anthropic reports none separately, so
the response's output stands in), and agentlib's context estimate counts that, not
the encrypted characters.

Compaction summaries read the reasoning's readable text and leave out signatures and
encrypted content. Readable text is often off by default; to get it, for summaries
or a UI:

```ts
providerOptions: {
  openai: {reasoningSummary: "auto"},
  anthropic: {thinking: {type: "adaptive", display: "summarized"}},
  google: {thinkingConfig: {includeThoughts: true}},
}
```

While streaming, `onChunk` receives the reasoning as `reasoning-delta` events.

## When reasoning is rejected

agentlib doesn't retry or repair rejected reasoning; the run fails with the
provider's error, as with any rejected request. To repair a broken run, 
remove the corrupt or invalid reasoning.
