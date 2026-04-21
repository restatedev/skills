# Python OpenAI Agents SDK Integration Reference

## Installation

```bash
pip install restate-sdk[serde]
```

## Core Pattern

Use `DurableRunner.run(agent, message)` to run an agent durably. Decorate tool functions with `@durable_function_tool` and use `restate_context()` inside tools for durable steps.

```python
import restate
from agents import Agent
from pydantic import BaseModel
from restate.ext.openai import restate_context, DurableRunner, durable_function_tool

class WeatherPrompt(BaseModel):
    message: str = "What is the weather in San Francisco?"

# TOOL
@durable_function_tool
async def get_weather(city: str) -> dict:
    """Get the current weather for a given city."""

    # Do durable steps using the Restate context
    async def call_weather_api(city: str) -> dict:
        return {"temperature": 23, "description": "Sunny and warm."}

    return await restate_context().run_typed(
        f"Get weather {city}", call_weather_api, city=city
    )


# AGENT
weather_agent = Agent(
    name="WeatherAgent",
    instructions="You are a helpful agent that provides weather updates.",
    tools=[get_weather],
)


# AGENT SERVICE
agent_service = restate.Service("agent")


@agent_service.handler()
async def run(_ctx: restate.Context, req: WeatherPrompt) -> str:
    # Runner that persists the agent execution for recoverability
    result = await DurableRunner.run(weather_agent, req.message)
    return result.final_output
```

## Key Requirements

- **Configure retry attempts** via `RunOptions` when needed.
- **Use `restate_context()`** actions for all steps inside tools.
- **Decorate tool functions with `@durable_function_tool`** to make them durable.
- The plugin executes tool calls always in sequence.

## Template

```bash
restate example python-openai-agents-template
```

## Durable Sessions

To add session management to the agent:

```python
chat = VirtualObject("Chat")


@chat.handler()
async def message(_ctx: ObjectContext, req: ChatMessage) -> dict:
    # Set use_restate_session=True to store the session in Restate's key-value store
    # Make sure you use a VirtualObject to enable this feature
    result = await DurableRunner.run(
        Agent(name="Assistant", instructions="You are a helpful assistant."),
        req.message,
        use_restate_session=True,
    )
    return result.final_output


@chat.handler(kind="shared")
async def get_history(ctx: restate.ObjectSharedContext):
    return await ctx.get("messages", type_hint=list[dict]) or []
```

## More Examples

`github.com/restatedev/ai-examples/openai-agents/`
