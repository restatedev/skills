# Track routing

Map the user's intake answer to exactly one track. When in doubt, pick the simpler one and offer the other as a follow-up.

## Tracks

### Quickstart — hello world

Pick when the user says any of:

- "Just show me" / "I want to see it work" / "five-minute demo"
- "Evaluate Restate" / "what does Restate feel like"
- They have no specific use case in mind.

Starter: `<lang>-hello-world` (one Greeter service). Reference: `quickstart.md`.

### Tour of Workflows

Pick when the user describes a **single long-running process** that needs to survive failures. Signals:

- Signup, onboarding, order processing, payment flow, booking, reservation
- "Background job", "cron-like but resilient", "long-running"
- "Retry indefinitely", "durable timers", "wait for user confirmation"
- "Saga" or "compensation" where the compensation sits in the same workflow
- "I want state that lives with the workflow instance"

Starter: `<lang>-tour-of-workflows`. Reference: `tour-of-workflows.md`.

### Tour of Microservice Orchestration

Pick when the user describes **multiple services that need to talk durably**. Signals:

- "Coordinate services", "service-to-service calls that must not get lost"
- "Replace my Temporal / Step Functions / Airflow"
- "Idempotency between services", "exactly-once between my services"
- "Virtual objects" / "stateful entities with concurrency control"
- "Webhook-driven", "awakeables", "external events from another system"
- "Saga across services" where compensations call other services

Starter: `<lang>-tour-of-orchestration`. Reference: `tour-of-orchestration.md`.

### AI agent on-ramp

Pick when the user mentions any of:

- "Agent", "LLM calls", "OpenAI / Anthropic / model calls"
- "Tool-using agent", "agent with memory", "agent loop"
- A specific framework: Vercel AI SDK, OpenAI Agents, Google ADK, Pydantic AI

No starter is cloned from `restate example` for this track. Reference: `ai-agents.md` — it routes to `restatedev/ai-examples` and the framework-specific references in the sibling `building-restate-services` skill.

## Disambiguation protocol

If the answer is ambiguous, ask **at most one** follow-up question. Examples:

- *"Is this one long-running process, or several services that need to coordinate?"* → chooses between Workflows and Orchestration.
- *"Do you already know which agent framework you'd use, or do you want a general Restate intro first?"* → chooses between AI on-ramp and Quickstart.

After the follow-up, commit to a track. Don't keep asking.

## Overlap rules

- If the user mentions "agent" AND one of the other tracks: start with the AI on-ramp. Workflows/orchestration patterns come up naturally inside it.
- If the user mentions "workflow" AND "microservices": start with the Tour of Workflows (simpler) and offer the orchestration tour at the end.
- If the user says they're brand new AND gives a concrete use case: run the Quickstart first (10 minutes), then the matching tour.
