# Python Pydantic AI Integration Reference

## Installation

```bash
pip install restate-sdk[serde]
```

## Core Pattern

Wrap a Pydantic AI `Agent` with `RestateAgent(agent)` for durable execution. Use `restate_context()` inside tool functions for durable step.

```python
import restate
from pydantic import BaseModel
from pydantic_ai import Agent, RunContext
from restate.ext.pydantic import RestateAgent, restate_context


class WeatherPrompt(BaseModel):
    message: str = "What is the weather in San Francisco?"

# AGENT
weather_agent = Agent(
    "openai:gpt-4o-mini",
    system_prompt="You are a helpful agent that provides weather updates.",
)

@weather_agent.tool()
async def get_weather(_run_ctx: RunContext[None], city: str) -> dict:
    """Get the current weather for a given city."""

    # Do durable steps using the Restate context
    async def call_weather_api(city: str) -> dict:
        return {"temperature": 23, "description": "Sunny and warm."}

    return await restate_context().run_typed(
        f"Get weather {city}", call_weather_api, city=city
    )

# AGENT SERVICE
restate_agent = RestateAgent(weather_agent)
agent_service = restate.Service("agent")


@agent_service.handler()
async def run(_ctx: restate.Context, req: WeatherPrompt) -> str:
    result = await restate_agent.run(req.message)
    return result.output
```

## Key Requirements

- **`RestateAgent` wraps the Pydantic AI agent** for durable execution.
- **Use `restate_context()`** actions for all durable steps inside tool functions.
- The plugin executes tool calls always in sequence.

## Template

```bash
restate example python-pydantic-ai-template
```

## Durable Sessions

To add session management to the agent: 

```python
agent = Agent(
    "openai:gpt-4o-mini",
    system_prompt="You are a helpful assistant.",
)
restate_agent = RestateAgent(agent)

chat = VirtualObject("Chat")


@chat.handler()
async def message(ctx: ObjectContext, req: ChatMessage) -> str:
    # Load message history from Restate's durable key-value store
    history = await ctx.get("messages", serde=MessageSerde())

    result = await restate_agent.run(req.message, message_history=history)

    # Store updated history back in Restate state
    ctx.set("messages", result.all_messages(), serde=MessageSerde())
    return result.output


@chat.handler(kind="shared")
async def get_history(ctx: restate.ObjectSharedContext) -> dict:
    return await ctx.get("messages", type_hint=dict) or {}
```

## More Examples

`github.com/restatedev/ai-examples/pydantic-ai/`