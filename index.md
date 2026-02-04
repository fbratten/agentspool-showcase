---
layout: default
title: Home
nav_order: 1
description: "agentspool - Inter-agent communication and coordination for LLM-powered AI entities"
permalink: /
---

# agentspool

> Inter-agent communication and coordination for LLM-powered AI entities

{: .fs-6 .fw-300 }

[Get Started](./docs/agent-integration.md){: .btn .btn-primary .fs-5 .mb-4 .mb-md-0 .mr-2 }
[View on GitHub](https://github.com/fbratten/agentspool){: .btn .fs-5 .mb-4 .mb-md-0 }

---

## What is agentspool?

A standalone Python library and CLI enabling AI agents, assistants, and bots to communicate and coordinate with each other. Built for the [AdaptiveArts.ai](https://adaptivearts.ai) ecosystem.

### The Problem

AI agents operate in isolated contexts. By default, there's no mechanism for agents to communicate with each other.

### The Solution

**agentspool** provides a SQLite-backed message spool with:

- **Atomic claim/lease semantics** for reliable delivery
- **Agent registry** with capability discovery
- **CLI tools** for send/poll/ack workflows
- **MCP server** with 14 tools for programmatic access
- **HTTP relay** for cross-device communication

## Architecture

```
Agent A ←→ Message Spool ←→ Agent B
   ↓             ↓             ↓
 send()    coordination.db   poll()
 poll()                      send()
```

## Key Features

| Feature | Description |
|:--------|:------------|
| **MessageV2 Protocol** | Typed payloads, priority levels, conversation threading |
| **SQLite Spool** | WAL-mode durable queue with atomic claim semantics |
| **HTTP Relay** | HMAC-SHA256 authenticated cross-device communication |
| **MCP Server** | 14 tools via FastMCP for AI agent integration |
| **Bridge System** | OpenClaw gateway integration for bot agents |

## Quick Start

```bash
# Install
pip install agentspool

# Register agents
python3 -m agent_comm register my-agent -c "code,research" -d local

# Send a message
python3 -m agent_comm send sender recipient "Hello!" -s "Greeting"

# Poll for messages
python3 -m agent_comm poll recipient
```

## Documentation

- [AI Agent Integration Guide](./docs/agent-integration.md) - Complete setup guide
- [Implementation Guide](./docs/implementation-guide.md) - Technical deep dive

## Related Projects

| Project | Role |
|:--------|:-----|
| [SPINE](https://fbratten.github.io/spine-showcase/) | Multi-agent backbone |
| [Minna Memory](https://fbratten.github.io/spine-showcase/docs/minna-memory-integration.html) | Agent registry + shared context |

---

*Part of the [AdaptiveArts.ai](https://adaptivearts.ai) ecosystem*
