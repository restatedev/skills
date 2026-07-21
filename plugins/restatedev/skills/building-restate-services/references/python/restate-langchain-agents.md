# Python LangChain Integration Reference

Use the Restate Python SDK 1.x integration with LangChain and LangGraph 1.x. Before upgrading an existing project across a major version, inspect its dependency constraints and migration guides.

## Installation

Start from the maintained template:

```bash
restate example python-langchain-template
```

The maintained template currently targets Python 3.13 or newer.

For an existing project, install the Restate serialization extra, LangChain, LangGraph, and the chosen model provider:

```bash
pip install "restate-sdk[serde]>=1.0,<2" "langchain>=1,<2" "langgraph>=1,<2" langchain-openai
```

## Core pattern

Attach `RestateMiddleware()` to a `create_agent` agent and invoke the agent from inside a Restate handler. Inside tools, obtain the active context with `restate_context()` and wrap side effects in `run_typed()`.

```python
import restate
from langchain.agents import create_agent
from langchain_core.messages import AnyMessage
from langchain_core.tools import tool
from langchain.chat_models import init_chat_model
from pydantic import BaseModel
from restate.ext.langchain import RestateMiddleware, restate_context


class WeatherPrompt(BaseModel):
    message: str = "What is the weather in San Francisco?"


# TOOL
@tool
async def get_weather(city: str) -> dict:
    """Get the current weather for a given city."""

    async def call_weather_api() -> dict:
        return {"temperature": 23, "description": "Sunny and warm."}

    # Durable step: results are journaled, so on retry we replay the value
    # rather than re-hitting the API.
    return await restate_context().run_typed(f"Get weather {city}", call_weather_api)


# AGENT
weather_agent = create_agent(
    model=init_chat_model("openai:gpt-5.4"),
    tools=[get_weather],
    system_prompt="You are a helpful agent that provides weather updates.",
    middleware=[RestateMiddleware()],
)


# AGENT SERVICE
agent_service = restate.Service("agent")


@agent_service.handler()
async def run(_ctx: restate.Context, req: WeatherPrompt) -> str:
    result = await weather_agent.ainvoke(
        {"messages": [{"role": "user", "content": req.message}]}
    )
    return result["messages"][-1].content
```

## What the middleware guarantees

- `RestateMiddleware` journals each model response so recovery replays it instead of calling the model again.
- It linearizes a batch of LangChain tool calls into a deterministic order for Restate replay.
- It does **not** journal the tool body. Wrap every external call or side effect inside a tool with `restate_context().run_typed()`.
- It must execute inside a Restate handler. Calling the agent without an active Restate context fails.

Do not add native `asyncio.gather()` around tool or model calls. Use Restate concurrency primitives for parallel workflows outside the LangChain agent loop.

## Retries and terminal failures

Use `RunOptions` on the middleware to bound model-call retries. Configure expensive or fragile tool steps separately:

```python
from langchain.agents import create_agent
from langchain.chat_models import init_chat_model
from restate import RunOptions
from restate.ext.langchain import RestateMiddleware

agent = create_agent(
    model=init_chat_model("openai:gpt-5.4"),
    tools=[get_weather],
    middleware=[RestateMiddleware(run_options=RunOptions(max_attempts=3))],
)
```

When a context action exhausts its retries, let `restate.TerminalError` propagate out of the tool and agent unless the handler has an explicit fallback. Do not convert Restate suspension or terminal failures into ordinary messages for the model.

## Durable sessions

Use a Virtual Object keyed by conversation ID and store typed LangChain messages in Restate state:

```python
chat = restate.VirtualObject("Chat")

agent = create_agent(
    model=init_chat_model("openai:gpt-5.4"),
    system_prompt="You are a helpful assistant.",
    middleware=[RestateMiddleware()],
)


@chat.handler()
async def message(ctx: restate.ObjectContext, req: ChatMessage) -> str:
    history = await ctx.get("messages", type_hint=ChatHistory) or ChatHistory()
    history.messages.append(HumanMessage(content=req.message))

    result = await agent.ainvoke({"messages": history.messages})

    ctx.set("messages", ChatHistory(messages=result["messages"]))
    return result["messages"][-1].content


@chat.handler(kind="shared")
async def get_history(ctx: restate.ObjectSharedContext) -> ChatHistory:
    return await ctx.get("messages", type_hint=ChatHistory) or ChatHistory()
```

Preserve the full LangChain message types and tool-call metadata when storing history. Bound long-running histories with a durable summarization or archival step.

## Common pitfalls

- Adding `RestateMiddleware` journals model calls, not arbitrary code in tools.
- Invoking `agent.invoke()` uses the synchronous path. Use `await agent.ainvoke(...)` inside an asynchronous Restate handler.
- Framework built-in tools can perform hidden I/O. Confirm that each one is journaled or replace it with an explicit durable tool.
- A LangGraph checkpointer and Restate Virtual Object state are separate state systems. Choose an intentional source of truth instead of accidentally persisting the same conversation twice.
- Returning `result["messages"][-1].content` may not be a string for every model or structured-output configuration. Match the handler output type to the configured agent response.

## More examples

- Tour and templates: https://github.com/restatedev/ai-examples/tree/main/langchain-python
- LangChain documentation: https://docs.restate.dev/ai/sdk-integrations/langchain
- Framework-neutral Restate agent guidance: `references/ai-agents.md`
