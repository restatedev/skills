# TypeScript Vercel AI SDK Integration Reference

## Installation

```bash
npm install @restatedev/vercel-ai-middleware ai @ai-sdk/openai
```

## Basic Example

Use the Restate middleware to wrap the language model:

```ts
import * as restate from "@restatedev/restate-sdk";
import { durableCalls } from "@restatedev/vercel-ai-middleware";
import { openai } from "@ai-sdk/openai";
import { generateText, stepCountIs, tool, wrapLanguageModel } from "ai";
import { z } from "zod";

// TOOL
async function getWeather(ctx: restate.Context, city: string) {
  // Do durable steps using the Restate context
  return ctx.run(`get weather ${city}`, () => {
    // Simulate calling the weather API
    return {temperature: 23, description: `Sunny and warm.`}
  })
}

// AGENT
const run = async (ctx: restate.Context, { prompt }: { prompt: string }) => {
  const model = wrapLanguageModel({
    model: openai("gpt-4o"),
    // Persist LLM responses
    middleware: durableCalls(ctx, { maxRetryAttempts: 3 }),
  });

  const { text } = await generateText({
    model,
    system: "You are a helpful agent that provides weather updates.",
    prompt,
    tools: {
      getWeather: tool({
        description: "Get the current weather for a given city.",
        inputSchema: z.object({ city: z.string() }),
        execute: async ({ city }) => getWeather(ctx, city),
      }),
    },
    stopWhen: [stepCountIs(5)],
    providerOptions: { openai: { parallelToolCalls: false } },
  });

  return text;
};

// AGENT SERVICE
const agent = restate.service({
  name: "agent",
  handlers: {
    run: restate.createServiceHandler({
      input: restate.serde.schema(z.object({
        prompt: z.string().default("What's the weather in San Francisco?"),
      })),
    }, run),
  },
});

restate.serve({ services: [agent] });
```

## Key Requirements

- **Set run options on `durableCalls`** to prevent infinite LLM retries.
- **Set `providerOptions: { openai: { parallelToolCalls: false } }` on generateText** -- parallel tool calls break replay.
- **Wrap all side effects in context actions inside tools.**
- **Streaming is not supported** -- `durableCalls` waits for the full response before journaling.

## Template

```bash
restate example typescript-vercel-ai-template
```

## Durable Sessions

To add session management to the agent:

```typescript
const chatAgent = restate.object({
  name: "Chat",
  handlers: {
    message: restate.createObjectHandler(
      { input: schema(ChatMessageSchema) },
      async (ctx: restate.ObjectContext, { message }: { message: string }) => {
        const model = wrapLanguageModel({
          model: openai("gpt-4o"),
          middleware: durableCalls(ctx, { maxRetryAttempts: 3 }),
        });

        // Retrieve the state
        const messages =
          (await ctx.get<ModelMessage[]>("messages", superJson)) ?? [];
        messages.push({ role: "user", content: message });

        const res = await generateText({
          model,
          system: "You are a helpful assistant.",
          messages,
        });

        // Update the state
        ctx.set("messages", [...messages, ...res.response.messages], superJson);
        return { answer: res.text };
      },
    ),
    // Shared handler to retrieve the history
    getHistory: shared(async (ctx: restate.ObjectSharedContext) =>
      ctx.get<ModelMessage[]>("messages", superJson),
    ),
  },
});
```

## More Examples

`github.com/restatedev/ai-examples/vercel-ai/`
