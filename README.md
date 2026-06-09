# WCP AI Build — Agent

> Design a WCP host agent — platform-agnostic design, platform-specific build.

A WCP **agent** is a native process running directly on the host machine. It exposes a
local HTTP API that widget containers reach via `host.docker.internal`. Unlike widgets,
agents run outside Docker and have direct access to host resources.

This repository covers the **platform-agnostic design phase**: what the agent does, what
it exposes, and how it communicates. Once the platform is confirmed, you are routed to the
platform-specific build skill.

---

## Quick Start

Give your AI a single instruction:

> _"Read https://github.com/penrithbeacon/wcp-ai-build-agent/blob/main/AI-SKILL.md
> and help me build a WCP agent."_

Your AI will establish the agent's purpose and API design, ask which platform(s) it will
run on, and hand off to the appropriate platform-specific skill.

---

## Supported Platforms

| Platform | Status | AI Skill |
|----------|--------|---------|
| macOS (Intel + Apple Silicon) | ✅ Available | [wcp-ai-build-agent-mac](https://github.com/penrithbeacon/wcp-ai-build-agent-mac/blob/main/AI-SKILL.md) |
| Linux | 🔜 Coming soon | — |
| Windows | 🔜 Coming soon | — |

---

## Links

- [WCP AI Build](https://github.com/penrithbeacon/wcp-ai-build) — start here if you're not sure what to build
- [WCP AI Build — macOS Agent](https://github.com/penrithbeacon/wcp-ai-build-agent-mac)
- [Widget Context Protocol](https://widgetcontextprotocol.com)
- [Penrith Beacon](https://penrithbeacon.com)
