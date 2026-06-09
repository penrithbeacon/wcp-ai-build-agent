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

## 2. Technology and Platform Negotiation

Follow the **Technology Negotiation Pattern** defined in
[wcp-ai-build AI-SKILL.md Section 2](https://github.com/penrithbeacon/wcp-ai-build/blob/main/AI-SKILL.md).

For agents, there are two interlinked decisions: **platform** and **language/runtime**.
Resolve platform first — language options differ by platform.

**Platform options (ask the developer):**
- macOS (Intel x86_64 and/or Apple Silicon ARM64)
- Linux (x86_64, ARM64)
- Windows (x86_64)
- Multiple platforms (common for agents intended for broad distribution)

Once the platform is confirmed, route to the platform-specific skill (Section 4). The
platform skill handles the language/runtime negotiation for that platform.

---

## 3. Skill: Design the Agent

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
   `GET /health` → `{ "status": "ok", "name": "...", "version": "..." }`

2. **Data endpoints** — one or more `GET` endpoints returning JSON data to the widget.
   For each, define: path, what it returns, how it is obtained (read file, run command,
   call OS API, query local DB, etc.)

3. **Control endpoints** — optional `POST` endpoints if the widget needs to trigger
   host actions. For each: path, what action it performs, what it returns.

4. **Persistent state** — does the agent need to remember configuration or state between
   restarts? If yes: a config file or local database (path under the user's home directory
   or a well-known config directory).

5. **Auto-start** — required for production use. The agent must start automatically at
   user login. The mechanism is platform-specific (Section 4).

### Phase D — Confirm outputs

Before routing to the platform skill, confirm:
- Agent name (kebab-case, e.g. `my-widget-agent`)
- Purpose summary (1–3 sentences)
- Full endpoint list with descriptions
- Communication port
- Persistent state requirements (or "none")
- Platform(s)

---

## 4. Platform Resolution

Route to the platform-specific skill based on the developer's answer:

| Platform | Action |
|----------|--------|
| **macOS** | Read [wcp-ai-build-agent-mac AI-SKILL.md](https://github.com/penrithbeacon/wcp-ai-build-agent-mac/blob/main/AI-SKILL.md) and follow it. Pass the Phase D outputs as context. |
| **Linux** | Platform skill coming soon (`wcp-ai-build-agent-linux`). Design decisions from Phases A–C are complete; implementation can proceed using standard Linux daemon patterns (systemd service). |
| **Windows** | Platform skill coming soon (`wcp-ai-build-agent-windows`). Design decisions from Phases A–C are complete; implementation can proceed using Windows Service or Task Scheduler patterns. |
| **Multiple platforms** | Route to each platform skill in turn. The Phase A–C design decisions carry forward unchanged — only the packaging and auto-start mechanism differs per platform. |

---

## 5. What Carries Forward to the Platform Skill

The platform skill will ask for these outputs. Have them ready:

| Item | Example |
|------|---------|
| Agent name | `my-widget-agent` |
| Purpose (1–3 sentences) | "Reads local project metadata and exposes it to the My Widget container." |
| Endpoint list | `GET /health`, `GET /projects`, `GET /project/{id}` |
| Communication port | `3749` |
| Persistent state | "Config file at `~/.my-widget-agent/config.json`" or "None" |
| Platform(s) confirmed | "macOS (Apple Silicon)" |
