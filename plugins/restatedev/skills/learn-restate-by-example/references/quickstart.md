# Quickstart — hello world

A 10-minute walkthrough of a Greeter service. Based on `https://docs.restate.dev/quickstart`. The goal is for the user to *see* a handler crash and transparently resume.

## Starter template

Run in an empty directory:

| Language | Command |
|---|---|
| TypeScript | `restate example typescript-hello-world` |
| Python | `restate example python-hello-world` |
| Java | `restate example java-hello-world-gradle` (or `java-hello-world-maven-spring-boot` if the user wants Spring Boot) |
| Go | `restate example go-hello-world` |

`cd` into the created directory before continuing.

## Stage 1 — Read the handler

Open the single service file (path varies by language — find it with `ls` or by reading `README.md`). Point out:

- The handler is an ordinary function that takes a `ctx` plus input.
- The SDK binds it to a service name and a handler name.
- That's the entire service surface — no controller, no queue, no DB client on the hot path.

Don't paste the code; read the file and cite lines. Ask the user if anything in the handler looks unfamiliar.

## Stage 2 — Run the service

Follow the starter's `README.md` run command. Typical:

- TS: `npm install && npm run app-dev` (or `bun run app-dev`).
- Python: `uv sync && uv run ./my_service.py` (exact file name in `README.md`).
- Java: `./gradlew run` (or `mvn spring-boot:run`).
- Go: `go run .`.

The service listens on `localhost:9080` by default. Leave it running.

## Stage 3 — Register the service with Restate

In a **new terminal**:

```bash
restate deployments register http://localhost:9080
```

Confirm the prompt. The server now knows about the Greeter service. List what's registered:

```bash
restate services list
```

Open `http://localhost:9070` in a browser and show the admin UI — the Greeter should be listed.

## Stage 4 — Invoke it

Via the playground: in the admin UI, click the Greeter service → handler → *Playground* → *Send*.

Via curl (equivalent):

```bash
curl -s localhost:8080/Greeter/greet -H 'Content-Type: application/json' -d '"Alice"'
```

Expect `"You said hi to Alice!"` (or similar — check the starter's handler to predict the exact output).

Explain: the request went to the Restate server (`:8080`), which appended to a journal and forwarded to the service. The response came back through the server.

## Stage 5 — Crash and resume

This is the point of the quickstart. Walk the user through adding a forced crash **before** the final step in the handler.

1. Edit the handler: add a `ctx.run("log", () => console.log("step 1 done"))` (or the language equivalent) near the top, then **after** it add `throw new Error("boom")` (or language equivalent). Save.
2. The starter usually hot-reloads. If not, restart the service.
3. Re-invoke:

   ```bash
   curl -s localhost:8080/Greeter/greet -H 'Content-Type: application/json' -d '"Alice"'
   ```

   The curl hangs. In the service logs: repeated retries. Each retry re-runs the handler. Point out: the `step 1 done` log appears **only once** — on replay, the already-journaled `ctx.run` returns the recorded value without re-executing.

4. Show the invocation as stuck:

   ```bash
   restate invocations list
   ```

   Or in the admin UI, open the invocation journal and show the entries.

5. Remove the `throw`. Save. The service hot-reloads, the next retry succeeds, the curl call returns.

This is the one stage worth lingering on. The user should leave with an intuitive picture of: *journal → replay → deterministic resume*.

## Stage 6 — Handoff

Offer the user the next step:

- *"Want to see how this scales to a full signup workflow with state, timers, and sagas?"* → Tour of Workflows (`tour-of-workflows.md`).
- *"Want to see how multiple services coordinate with virtual objects and awakeables?"* → Tour of Microservice Orchestration (`tour-of-orchestration.md`).
- *"Want to build your own?"* → hand off to the sibling `building-restate-services` skill.

Always mention: the `restate-docs` MCP server answers deeper questions (deployment, Kafka, server config) — they don't need to go to the website.
