# Tools

Lessons:
[02_tools.ts](https://github.com/restatedev/agentlib/blob/main/packages/examples/src/lessons/02_tools.ts),
[10_service_tool.ts](https://github.com/restatedev/agentlib/blob/main/packages/examples/src/lessons/10_service_tool.ts),
[11_coding_agent.ts](https://github.com/restatedev/agentlib/blob/main/packages/examples/src/lessons/11_coding_agent.ts),
[14_web_search.ts](https://github.com/restatedev/agentlib/blob/main/packages/examples/src/lessons/14_web_search.ts),
[17_mcp_tools.ts](https://github.com/restatedev/agentlib/blob/main/packages/examples/src/lessons/17_mcp_tools.ts)

Tools are a map on the agent (`tools: {name: tool}`) or on a single run
(`run(input, {tools})`). The key is the name the model sees. Inputs are Zod
schemas; the SDK validates the model's arguments before `execute` runs.

## asyncTool: an ordinary async call

```ts
import {asyncTool, ToolError} from "@restatedev/agentlib";

lookupTicket: asyncTool({
  description: "Look up a ticket's status.",
  input: z.object({ticketId: z.string()}),
  retry: {maxAttempts: 5, initialInterval: {milliseconds: 250}, exponentiationFactor: 2},
  async execute({ticketId}, {signal, callId, context}) {
    const response = await fetch(`https://tickets.example/${ticketId}`, {signal});
    if (response.status === 404) throw new ToolError(`No ticket ${ticketId}`, "not_found");
    return (await response.json()) as {ticketId: string; status: string};
  },
}),
```

The result is journaled once `execute` succeeds. It must be JSON: cast
`response.json()`, which is `unknown`, to the shape you return.

- A thrown `Error` retries under `retry`; the default is three attempts.
- A `TerminalError` stops retrying. Exhausted retries end the same way.
- A `ToolError` is a `TerminalError` with a code for the model, such as
  `not_found`.
- In every one of these cases the model receives
  `{status: "error", code, message}`, with the `ToolError`'s code or
  `tool_failed`, and the run continues.
- To fail the whole run instead, for example when the request it serves is
  no longer valid, throw `AgentTerminalError`. Every agent run that encloses
  this one fails too.
- Use `retry: {maxAttempts: 1}` for actions that must not repeat, such as
  running a shell command.

`error-handling.md` covers the same rules for hooks, sub-agents, model calls,
and handlers a tool calls.

## tool: compose Restate operations

`tool` has a generator `execute`, so it can use anything from the gen SDK:
RPC, `restate.run`, sleeps, signals, and state.

```ts
import {tool} from "@restatedev/agentlib";

reserve: tool({
  description: "Reserve stock for an order.",
  input: z.object({sku: z.string(), quantity: z.number().int().positive()}),
  *execute({sku, quantity}, {context}) {
    const ok = yield* restate.client(inventory, sku).reserve(quantity);
    if (!ok) throw new ToolError(`Not enough ${sku}`, "out_of_stock");
    return {sku, quantity, order: context.orderId};
  },
}),
```

Code in the generator body re-runs on replay. Put external I/O in
`restate.run` or `asyncTool`.

## Restate handlers as tools

```ts
import {handlerTool, serviceTool} from "@restatedev/agentlib";

// One handler, with the arguments you choose:
sendEmail: handlerTool({
  description: "Send an email.",
  input: z.object({to: z.string(), body: z.string()}),
  call: (input) => restate.client(emails).send(input),
}),

// A whole service from its interface: the model picks the handler.
// Descriptions and input schemas come from restate.iface.schemas.
refund: serviceTool(refundIface),
```

Each call is a durable child invocation. Interrupting the run cancels it.
A handler that fails with a terminal error is reported to the model as a tool
error, with the code of a `ToolError`; a handler that throws
`AgentTerminalError` fails the run instead. Pass
`{iface, handler: "issueRefund"}` to scope `serviceTool` to one handler.
Objects and workflows also take a `key`. Configure retries with
`retryPolicy` on the called handler.

`yield* restateTools({adminUrl, allow})` from `@restatedev/agentlib/discovery`
runs inside a handler and builds tools from every registered handler
annotated with the metadata key `restate.dev/agent`. Filter them with
`allow(target, name)`; `token` supplies the admin API's bearer token. The result's `targets`
maps each tool name to its handler (`service`, `handler`, `keyed`), for
permissions keyed by `service/handler`.

## Per-request resources through context

A tool reads run `context` for things like sandboxes, clients, and tenant
data. Create the resource in a journaled step before the run and release it
in `finally` (lesson 11):

```ts
function* (task) {
  const sandbox = yield* connect();          // restate.run creates it once
  try {
    const result = yield* coder.run(task, {context: {sandbox}});
    if (result.status !== "completed") return result.output ?? result.reason;
    return result.output;
  } finally {
    yield* release(sandbox);                 // restate.run terminates it once
  }
}
```

Inside `agent<Ctx>({tools: {...}})`, `context` is typed from the agent. A
tool defined on its own needs the type on its variable, or `context` is
`unknown`:

```ts
const readFile: Tool<Ctx> = asyncTool({
  description: "Read a text file.",
  input: z.object({path: z.string()}),
  execute: ({path}, {context}) => context.sandbox.filesystem.readText(path),
});
```

Lesson 14 defines a Tavily search tool locally with `asyncTool` and `fetch`.
Set `TAVILY_API_KEY` on the example server. There is no library web-search adapter.

## Remote and built-in tools

```ts
import {mcpTools} from "@restatedev/agentlib/mcp";

// Inside a handler: the catalog is fetched once and journaled.
const docs = yield* mcpTools({
  url: "https://docs.restate.dev/mcp",
  namespace: "docs", // tools become docs_<name>
  protocol: "stateless",
  deferred: false, // default true: hidden until searched
  allow: (name) => name !== "submit_feedback",
});
const result = yield* guide.run(question, {tools: docs.tools});
```

`mcpTools` also takes `token` (a generator returning a bearer token),
`protocol: "stateful"` for servers that need sessions, `timeoutMs` per request
(default 60000), `onAuthorizationRequired` to wait for fresh credentials, and
`retry` for tool calls (default three attempts; each call sends an `Idempotency-Key`, but a
server that ignores it may repeat work). A credential the server echoes back
is replaced with `[redacted]` before any result or error is journaled. Call
`close()` when done with a stateful session. A namespace is 1 to 24 letters,
digits, `_` or `-`; anything else is a terminal error.

Every built-in tool (`sleepTool`, `humanApprovalTool`, `cancelOperationTool`,
`searchToolsTool`, `programTool`) accepts `description` and
`describe` to replace its model-facing description and its display info.

## Large catalogs and advanced tool calling

**Tool search and code mode are on by default.** When the agent or the run
has a `deferred` tool, the run adds `search_tools`, and the model sees a
deferred tool's schema only after a search loads it. When the agent or the run
has a tool a program can call (any tool except a handoff or a search tool),
the run adds `run_program` for code mode (programmatic tool calling): the
model writes one JavaScript program that calls several tools, and only the
program's result enters the model's context. Programs run in a QuickJS sandbox
without network or filesystem access.

```ts
const assistant = agent({
  model,
  tools: {lookupOrder, refund},
  toolSearch: false, // list deferred tools from the start
  codeMode: false, // leave run_program out
});
```

- A search tool or program tool in `tools` replaces the default. For example,
  `program: programTool({tools: ["lookupOrder"], limits: {maxToolCalls: 20}})`
  limits which tools a program can call and how many calls it makes.
- Calls a program makes go through the same guardrails and approvals as direct
  calls, and count toward `maxToolCalls`.
- Like `read_tool_result`, both tools belong to the run. After a handoff, the
  run adds them based on the new agent's tools.
- A tool of your own named `search_tools` or `run_program` fails the run when
  the run would add that default. Rename the tool, or turn the default off.

| Option                                                            | Effect                                                                                                       |
| ----------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| `deferred: true` on a tool                                        | The tool's schema is hidden until the model finds it with `search_tools`                                     |
| `background: true`                                                | The model gets a pending result and keeps working; the real result arrives in a later step                   |
| `sleepTool()`, `cancelOperationTool()`                            | Let the model wait durably, or cancel a background operation                                                 |
| `programTool({tools, limits})` (`/program`)                       | Replaces `run_program` for code mode; `tools` narrows which tools a program may call, and `limits` bounds its work |
| `describe` / `context.describe(info)`                             | Display info for UIs, delivered through progress events                                                      |
| `context.callTool(name, input)`                                   | Call another tool from a tool, under the same policy and budgets                                             |
