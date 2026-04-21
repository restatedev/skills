# Python Google ADK Integration Reference

## Installation

```bash
pip install restate-sdk[serde]
```

## Core Pattern

Add `RestatePlugin()` to the ADK App's plugins list. Use `restate_context()` Context actions inside tool functions for durable steps.

```python
import restate
from restate.ext.adk import RestatePlugin, restate_context
from google.adk import Runner
from google.adk.apps import App
from google.adk.sessions import InMemorySessionService
from google.genai.types import Content, Part
from google.adk.agents.llm_agent import Agent
from pydantic import BaseModel

class WeatherPrompt(BaseModel):
    user_id: str = "user-123"
    message: str = "What is the weather like in San Francisco?"


# TOOL
async def get_weather(city: str) -> dict:
    """Get the current weather for a given city."""
    # Do durable steps using the Restate context
    async def call_weather_api(city: str) -> dict:
        return {"temperature": 23, "description": "Sunny and warm."}

    return await restate_context().run_typed(
        f"Get weather {city}", call_weather_api, city=city
    )


# AGENT
# Specify your agent in the default ADK way
agent = Agent(
    model="gemini-2.5-flash",
    name="weather_agent",
    instruction="You are a helpful agent that provides weather updates.",
    tools=[get_weather],
)

APP_NAME = "agents"
app = App(name=APP_NAME, root_agent=agent, plugins=[RestatePlugin()])
session_service = InMemorySessionService()

# AGENT SERVICE + HANDLER
agent_service = restate.Service("agent")


@agent_service.handler()
async def run(ctx: restate.Context, req: WeatherPrompt) -> str | None:
    # Start new session
    session_id = str(ctx.uuid())
    session = await session_service.get_session(
        app_name=APP_NAME, user_id=req.user_id, session_id=session_id
    )
    if not session:
        await session_service.create_session(
            app_name=APP_NAME, user_id=req.user_id, session_id=session_id
        )

    # Run the durable agent
    runner = Runner(app=app, session_service=session_service)
    events = runner.run_async(
        user_id=req.user_id,
        session_id=session_id,
        new_message=Content(role="user", parts=[Part.from_text(text=req.message)]),
    )

    final_response = None
    async for event in events:
        if event.is_final_response() and event.content and event.content.parts:
            if event.content.parts[0].text:
                final_response = event.content.parts[0].text
    return final_response
```

## Key Requirements

- **`RestatePlugin()` handles durability** -- add it to the App's `plugins` list.
- **Configure retry attempts** via `RunOptions` when needed.
- **Use `restate_context()`** actions for all durable steps inside tool functions.
- The plugin executes tool calls always in sequence.

## Template

```bash
restate example python-google-adk-template
```

## Durable Sessions

To add session management to the agent:

```python
agent = Agent(
    model="gemini-2.5-flash",
    name="assistant",
    instruction="You are a helpful assistant. Be concise and helpful.",
)
app = App(name=APP_NAME, root_agent=agent, plugins=[RestatePlugin()])
runner = Runner(app=app, session_service=RestateSessionService())

chat = restate.VirtualObject("Chat")


@chat.handler()
async def message(ctx: restate.ObjectContext, req: ChatMessage) -> str | None:
    events = runner.run_async(
        user_id=ctx.key(),
        session_id=req.session_id,
        new_message=Content(role="user", parts=[Part.from_text(text=req.message)]),
    )
    return await parse_agent_response(events)


@chat.handler(kind="shared")
async def get_history(ctx: restate.ObjectSharedContext, session_id: str):
    return await ctx.get(f"session_store::{session_id}", type_hint=list[dict]) or []
```

## More Examples

`github.com/restatedev/ai-examples/google-adk/`