# WCP AI Build — Agent AI Skill

> **Version 1.0.0** | For consumption by any AI engine building a WCP host agent.
> Source of truth: [github.com/penrithbeacon/wcp-ai-build-agent](https://github.com/penrithbeacon/wcp-ai-build-agent)

---

## 1. What Is a WCP Agent

A WCP agent is a **native process running directly on the host machine** — not in Docker.

| Property | Agent | Widget |
|----------|-------|--------|
| Runtime environment | Host OS, native process | Docker container |
| Network address | `127.0.0.1` (loopback only) | Container port, reachable from dashboard |
| Reached by widgets via | `host.docker.internal:<port>` | N/A |
| Has a UI | No | Yes — HTML served into iframe cards |
| Access to host resources | Full — file system, OS APIs, installed tools | Only what is mounted into the container |
| Auto-start mechanism | OS-native (launchd / systemd / Task Scheduler) | Docker restart policy |

### What agents are used for

Agents bridge the gap between widget containers (which are sandboxed) and the host
machine's data and capabilities. Common use cases:
- Providing local system data (running processes, installed tools, resource usage)
- Reading host-side configuration files or databases
- Executing host commands on behalf of a widget
- Integrating with native OS APIs (keychain, notifications, accessibility)

### Reference example

The **CLAUD agent** ships with the Claude Analytics widget:
- Source: `agents/mac-arm64/` in [wcp-widget-claude](https://github.com/penrithbeacon/wcp-widget-claude)
- Provides: Claude Code session data, MCP server config, installed plugins, system info
- Consumed by: the Claude Analytics widget container via `host.docker.internal:3747`

---

## 2. Parallel Pipeline Awareness

This skill may be invoked in two ways:

**A — Standalone:** The developer wants to build an agent only (no widget). Follow this
skill linearly from Section 3 onwards.

**B — Parallel (most common):** This skill was invoked from within a widget pipeline
because the widget requires host filesystem or OS access. Both the widget and agent
pipelines are running simultaneously in the same conversation.

When running in parallel:
- Design decisions made in the widget pipeline carry into this pipeline and vice versa.
- **Integration convergence points** — the following questions involve both pipelines
  and must be answered together:
  - **Port assignment** — the agent's port must not conflict with the widget's port.
    Both ports are drawn from the same occupied port set.
  - **Data contract** — agree on the exact endpoint paths and response schemas the
    widget will call on the agent. Define these jointly so both sides implement the
    same interface.
  - **`host.docker.internal` reference** — the widget container calls the agent at
    `host.docker.internal:<agent-port>`. Confirm this address is used in the widget
    source (not `localhost`, which resolves to the container itself).
  - **Health check** — the widget detects agent availability at startup via
    `GET host.docker.internal:<agent-port>/health`. Agree on the response format.
- When a convergence point is reached, ask the questions that cover both pipelines
  together rather than separately.

**Companion agent pattern:**

A companion agent is one that ships alongside a specific widget and is primarily designed
to serve that widget. Characteristics:
- Named after the widget: `wcp-agent-<widget-name>` or `<widget-name>-agent`
- The widget's Settings component hosts the agent installer download
  (served from the widget container at `GET /widget/agent/installer`)
- The widget degrades gracefully when the agent is not present, with a clear UI prompt
  in the Settings component to download and install it
- The agent installer is bundled inside the widget's Docker image

---

## 3. Technology and Platform Negotiation


Follow the **Technology Negotiation Pattern** defined in
[wcp-ai-build AI-SKILL.md Section 2](https://github.com/penrithbeacon/wcp-ai-build/blob/main/AI-SKILL.md).

For agents, there are two interlinked decisions: **platform** and **language/runtime**.
Resolve platform first — language options differ by platform.

**Platform options (ask the developer):**
- macOS (Intel x86_64 and/or Apple Silicon ARM64)
- Linux (x86_64, ARM64)
- Windows (x86_64)
- Multiple platforms (common for agents intended for broad distribution)

Once the platform is confirmed, route to the platform-specific skill (Section 5). The
platform skill handles the language/runtime negotiation for that platform.

---

## 4. Skill: Design the Agent

Establish the design intent before routing to a platform skill. Work through these phases:

### Phase A — Purpose

Ask:
1. What host data or capabilities does this agent provide?
2. Which widget(s) will consume it? (Or is it for a new widget being built alongside?)
3. Does anything similar already exist on this machine? (Avoid port conflicts and
   duplicate functionality.)

### Phase B — Communication

The standard WCP agent communication pattern is a **local HTTP API on loopback**:
- Listens on `127.0.0.1` only — never `0.0.0.0`
- Widget containers reach it via `host.docker.internal:<port>`
- This is intentionally isolated: only processes on the local machine can call it

Confirm:
1. Is the local HTTP API pattern appropriate? (Yes for almost all cases.)
2. Port number — must not conflict with widget container ports or other agents.
   Use the occupied port set recorded in wcp-ai-build Step 1 (from Bonjour query or
   developer-provided list). Do not assume any port is free — check against that set.
3. Does any endpoint need to accept data FROM the widget (POST), or is this a read-only
   data source (GET only)?

### Phase C — Capabilities

Define the agent's endpoint list:

1. **Health endpoint** — mandatory:
   `GET /health` → `{ "status": "ok", "name": "...", "version": "...", "platform": "..." }`

2. **Agent WCP manifest** — mandatory:
   `GET /agent/wcp` → JSON manifest describing the agent. Analogous to `GET /widget/wcp`
   for widgets. Must include:
   ```json
   {
     "name": "<agent-name>",
     "version": "<semver>",
     "platform": "<macOS|Linux|Windows>",
     "port": <port>,
     "endpoints": ["GET /health", "GET /agent/wcp", "..."],
     "companion_widget": "<wcp-widget-name>"
   }
   ```
   The `companion_widget` field is required for companion agents; omit for standalone agents.

3. **WCP logs endpoint** — mandatory:
   `GET /agent/logs` → WCP logs protocol envelope:
   ```json
   {
     "schema": "wcp-logs/1.0",
     "name": "<agent-name>",
     "entries": [
       { "ts": "<ISO8601Z>", "level": "info|warn|error", "msg": "..." }
     ]
   }
   ```
   Support `?limit=N`, `?level=info|warn|error`, `?since=<ISO8601Z>`.
   In-memory ring buffer, max 500 entries. Always returns 200.

4. **Bonjour registration** — mandatory startup behaviour:

   On startup, the agent must discover the Bonjour agent's actual port and register with
   it. The Bonjour agent's port is **not fixed** — do not hardcode it. Discover it from
   the port advertisement file:

   ```
   ~/Library/Application Support/penrithbeacon/bonjour-agent.port
   ```

   This file contains the Bonjour agent's actual bound port as a plain integer. Read it
   at registration time (not at startup) so the agent does not fail if Bonjour has not
   yet written the file.

   Once the port is known, register at:
   ```
   POST http://127.0.0.1:<bonjour-agent-port>/agent/register
   ```
   with body:
   ```json
   {
     "name": "<agent-name>",
     "port": <port>,
     "health": "/health",
     "companion_widget": "<wcp-widget-name>",
     "platform": "<platform>"
   }
   ```

   **Python pattern for port file discovery:**
   ```python
   import os, pathlib

   BONJOUR_PORT_FILE = pathlib.Path.home() / "Library" / "Application Support" \
                       / "penrithbeacon" / "bonjour-agent.port"

   def get_bonjour_port():
       try:
           return int(BONJOUR_PORT_FILE.read_text().strip())
       except Exception:
           return None
   ```

   Registration must be attempted in a **background thread** so it does not block agent
   startup. Use **exponential backoff retry** (suggested: up to 10 attempts, starting at
   2 seconds, doubling each time, max 60 seconds between attempts). The agent operates
   fully if the Bonjour agent is not running or the port file does not yet exist —
   registration failure is not fatal.

   > **Historical note:** early agents (e.g. the markdown editor companion agent before
   > Bonjour was built) hardcoded `127.0.0.1:3746` as a placeholder Bonjour address.
   > That was a beta placeholder. Any agent that still hardcodes a Bonjour port must be
   > updated to use the port file discovery pattern before it can leave beta.

5. **Data endpoints** — one or more `GET` endpoints returning JSON data to the widget.
   For each, define: path, what it returns, how it is obtained (read file, run command,
   call OS API, query local DB, etc.)

6. **Control endpoints** — optional `POST` endpoints if the widget needs to trigger
   host actions. For each: path, what action it performs, what it returns.

7. **Persistent state** — does the agent need to remember configuration or state between
   restarts? If yes: a config file or local database (path under the user's home directory
   or a well-known config directory).

8. **Auto-start** — required for production use. The agent must start automatically at
   user login. The mechanism is platform-specific (Section 5).

### Phase D — Confirm outputs

Before routing to the platform skill, confirm:
- Agent name (kebab-case, e.g. `my-widget-agent`)
- Purpose summary (1–3 sentences)
- Full endpoint list with descriptions
- Communication port
- Persistent state requirements (or "none")
- Platform(s)

---

## 5. Platform Resolution

Route to the platform-specific skill based on the developer's answer:

| Platform | Action |
|----------|--------|
| **macOS** | Read [wcp-ai-build-agent-mac AI-SKILL.md](https://github.com/penrithbeacon/wcp-ai-build-agent-mac/blob/main/AI-SKILL.md) and follow it. Pass the Phase D outputs as context. |
| **Linux** | Platform skill coming soon (`wcp-ai-build-agent-linux`). Design decisions from Phases A–C are complete; implementation can proceed using standard Linux daemon patterns (systemd service). |
| **Windows** | Platform skill coming soon (`wcp-ai-build-agent-windows`). Design decisions from Phases A–C are complete; implementation can proceed using Windows Service or Task Scheduler patterns. |
| **Multiple platforms** | Route to each platform skill in turn. The Phase A–C design decisions carry forward unchanged — only the packaging and auto-start mechanism differs per platform. |

---

## 6. Release Pipeline — End State

The platform skill (Section 5) is not the final step. When the platform skill completes
its build and test steps, it will hand off to the **wcp-ai-release pipeline**.

The full chain is:

```
wcp-ai-build → wcp-ai-build-agent → wcp-ai-build-agent-{platform} → wcp-ai-release
```

The release pipeline runs the pre-release audit (endpoint checks, Bonjour registration
verification, loopback binding verification), generates documentation (README.md,
specification.md), and publishes the agent installer to GitHub Releases.

**The agent build is not done until wcp-ai-release has passed its audit gate.**

---

## 7. What Carries Forward to the Platform Skill

The platform skill will ask for these outputs. Have them ready:

| Item | Example |
|------|---------|
| Agent name | `my-widget-agent` |
| Purpose (1–3 sentences) | "Reads local project metadata and exposes it to the My Widget container." |
| Endpoint list | `GET /health`, `GET /projects`, `GET /project/{id}` |
| Communication port | `3749` |
| Persistent state | "Config file at `~/.my-widget-agent/config.json`" or "None" |
| Platform(s) confirmed | "macOS (Apple Silicon)" |
| Companion widget name | `wcp-widget-<name>` (if this is a companion agent) |
| GitHub username + PAT | Carried from wcp-ai-build Step 1 |
| Credentials file path | Carried from wcp-ai-build Step 1 |
