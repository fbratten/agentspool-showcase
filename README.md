# agentspool Showcase

> Inter-agent communication and coordination for LLM-powered AI entities

**Live Site:** [https://fbratten.github.io/agentspool-showcase](https://fbratten.github.io/agentspool-showcase)
**Code Repository:** [https://github.com/fbratten/agentspool](https://github.com/fbratten/agentspool)

## Overview

agentspool enables AI agents, assistants, and bots to communicate and coordinate with each other. Built for the [AdaptiveArts.ai](https://adaptivearts.ai) ecosystem.

**Goal:** N-to-N cross-device agent communication with reliable message delivery.

## Architecture

```
                    Agent Registry (Minna Memory)
                    ┌────────────────────────────┐
                    │  Entity: agent:{name}       │
                    │  Attrs: capabilities, state │
                    └──────────┬─────────────────┘
                               │
           ┌───────────────────┼───────────────────┐
           │                   │                   │
    ┌──────┴──────┐    ┌──────┴──────┐    ┌──────┴──────┐
    │  Agent A    │    │  OpenClaw   │    │  Agent N    │
    │  (Device 1) │    │  Gateway    │    │  (any PC)   │
    └──────┬──────┘    └──────┬──────┘    └──────┬──────┘
           │                   │                   │
           └───────────────────┼───────────────────┘
                               │
              Transport Layer (SQLite → HTTP → MCP)
```

## Key Features

| Feature | Description |
|---------|-------------|
| **MessageV2 Protocol** | Typed payloads, priority levels, conversation threading, TTL |
| **SQLite Spool** | WAL-mode durable queue with atomic claim, lease/ack semantics |
| **HTTP Relay** | HMAC-SHA256 authenticated cross-device communication |
| **MCP Server** | 14 tools via FastMCP for AI agent integration |
| **Bridge System** | OpenClaw gateway integration for bot agents |

## Transport Layer

| Transport | Mode | Use Case |
|-----------|------|----------|
| `SQLiteTransport` | **Primary** | Local N-agent communication via coordination.db |
| `HTTPTransport` | **Cross-device** | HMAC-authenticated relay (PC-to-PC via Tailscale/LAN) |
| `FileTransport` | Debug | JSON files in inbox directories |

## MCP Server Tools

14 tools available via FastMCP:

| Category | Tools |
|----------|-------|
| **Messaging** | `comm_send`, `comm_poll`, `comm_ack`, `comm_nack` |
| **Registry** | `comm_register_agent`, `comm_discover_agents`, `comm_heartbeat`, `comm_deregister` |
| **Status** | `comm_message_status`, `comm_spool_stats`, `comm_get_conversation` |
| **Maintenance** | `comm_cleanup`, `comm_relay_gen_secret`, `comm_relay_list_secrets` |

## Quick Start

```bash
# Register agents
python3 -m agent_comm register agent-a -c "code,mcp,bash" -d device1
python3 -m agent_comm register agent-b -c "research,chat" -d device2

# Send a message
python3 -m agent_comm send agent-a agent-b "Research AI frameworks" \
    -s "Research task" --priority high

# Poll for messages (as recipient)
python3 -m agent_comm poll agent-b

# Acknowledge processing
python3 -m agent_comm ack msg_20260130_143022_a1b2c3 agent-b
```

## MessageV2 Protocol

```json
{
  "id": "msg_20260130_143022_a1b2c3",
  "version": "2.0",
  "from": "agent-a",
  "to": "agent-b",
  "subject": "Research task",
  "body": "Research AI agent frameworks and summarize findings.",
  "timestamp": "2026-01-30T14:30:22Z",
  "priority": "high",
  "routing": {
    "conversation_id": "conv_001",
    "ttl_hours": 24
  },
  "payload": {
    "type": "task_assignment",
    "data": { "scope": "broad", "max_sources": 5 }
  }
}
```

## Development Status

| Phase | Status | Description |
|-------|--------|-------------|
| **1. Foundation** | Complete | SQLite spool, Transport ABC, registry, CLI |
| **2. Gateway Bridge** | Complete | Bridge ABC, OpenClaw bridge, polling runner |
| **3. HTTP Relay** | Complete | FastAPI relay server, HMAC auth, HTTPTransport |
| **4. MCP Server** | Complete | 14 tools via FastMCP |

## Testing

- **212 pytest tests** covering all modules
- **28 script integration tests** for end-to-end workflows
- **Total: 240 tests**

## Related Projects

| Project | Relationship |
|---------|-------------|
| [SPINE](https://fbratten.github.io/spine-showcase/) | Multi-agent backbone (agent-comm-mcp is SPINE server #19) |
| [Minna Memory](https://fbratten.github.io/spine-showcase/docs/minna-memory-integration.html) | Agent registry + shared context |
| OpenClaw | Gateway runtime for bot agents |

## Documentation

- [Claude Code Integration Guide](./docs/claude-code-integration.md)
- [Implementation Guide](./docs/implementation-guide.md)

## License

MIT License - See [LICENSE](./LICENSE) for details.

---

*Part of the [AdaptiveArts.ai](https://adaptivearts.ai) ecosystem*
