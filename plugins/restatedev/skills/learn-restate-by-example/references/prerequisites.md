# Prerequisites

Before cloning a starter, check these are available. Install what's missing — but **ask before running install commands**.

Detect the OS via `uname -s` (`Darwin` = macOS, `Linux` = Linux, anything with `MINGW`/`MSYS`/`CYGWIN` = treat as Windows-under-Git-Bash; WSL reports `Linux`).

## 1. Restate CLI

**Check:**

```bash
restate --version
```

**Install if missing:**

- macOS (Homebrew): `brew install restatedev/tap/restate`
- npm (cross-platform): `npm install -g @restatedev/restate`
- Cargo: `cargo install --locked restate-cli`
- Or download a release binary from `https://github.com/restatedev/restate/releases` and put it on `$PATH`.

If the user doesn't have Homebrew/npm/cargo, prefer the release binary.

## 2. Restate Server

The server is a single Rust binary. Two run options — the user picks one.

**Option A — Docker (simplest):**

```bash
docker --version                              # check
docker run --rm --name restate_dev \
  -p 8080:8080 -p 9070:9070 -p 9071:9071 \
  docker.restate.dev/restatedev/restate:latest
```

Run this in a separate terminal (or with `-d` to detach). Ports: `8080` ingress, `9070` admin + UI, `9071` cluster.

**Option B — Native binary:**

- macOS: `brew install restatedev/tap/restate-server`
- Or download from the same releases page as the CLI.
- Start: `restate-server` (no args; runs with sensible defaults).

Do **not** start both options at once — they bind the same ports.

**Verify running:**

```bash
curl -s http://localhost:9070/health
```

Or open `http://localhost:9070` in a browser.

## 3. Language runtime

Ask only for the language the user picked in intake.

### TypeScript

```bash
node --version    # need >= 20
```

A package manager — any of `npm`, `pnpm`, `yarn`, `bun`.

Missing Node: recommend `nvm` (`curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash`), then `nvm install 20`.

### Python

```bash
python3 --version   # need >= 3.11
```

And `uv` (recommended) or `pip`. Install `uv`:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

### Java

```bash
java --version   # need >= 21
mvn --version    # OR gradle --version
```

Missing JDK: recommend `sdkman` (`curl -s https://get.sdkman.io | bash`), then `sdk install java 21-tem`.

### Go

```bash
go version   # need >= 1.23
```

Missing Go: `brew install go` on macOS, or download from `https://go.dev/dl/`.

## 4. curl

Used for hitting the service and the playground.

```bash
curl --version
```

Almost always pre-installed. If not, install via the system package manager.

## Report results

When done checking, give the user a short bulleted status:

- What's already present (with version).
- What's missing.
- For each missing item, the single install command you'd run.

Then ask: *"Shall I install the missing pieces, or will you?"*

If the user says "install it", run the commands one at a time and show their output.
