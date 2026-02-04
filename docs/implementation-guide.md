---
layout: default
title: Implementation Guide
nav_order: 3
description: "Technical deep dive into agentspool architecture and implementation"
---

# Implementation Guide

Technical deep dive into agentspool architecture, protocols, and implementation details.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [MessageV2 Protocol](#messagev2-protocol)
3. [Transport Layer](#transport-layer)
4. [Agent Registry](#agent-registry)
5. [MCP Server](#mcp-server)
6. [HTTP Relay](#http-relay)
7. [Security Model](#security-model)

---

## Architecture Overview

```
                    Agent Registry (Minna Memory or local JSON)
                    ┌────────────────────────────┐
                    │  Entity: agent:{name}       │
                    │  Attrs: capabilities, state │
                    └──────────┬─────────────────┘
                               │
           ┌───────────────────┼───────────────────┐
           │                   │                   │
    ┌──────┴──────┐    ┌──────┴──────┐    ┌──────┴──────┐
    │ Claude Code │    │   Nelly /   │    │  Agent N    │
    │  (WSL PC1)  │    │   Moltbot   │    │  (any PC)   │
    └──────┬──────┘    └──────┬──────┘    └──────┬──────┘
           │                   │                   │
           └───────────────────┼───────────────────┘
                               │
              Transport Layer (SQLite → HTTP → MCP)
```

### Project Structure

```
agentspool/
├── agent_comm/                   # Core library
│   ├── coordinator.py            # N-agent coordinator (hybrid routing)
│   ├── registry.py               # Agent registry (Minna + local fallback)
│   ├── message_types.py          # Pydantic: MessageV2, PayloadType, Priority
│   ├── spool.py                  # SQLite message spool
│   ├── transports/               # Transport ABC + implementations
│   ├── relay/                    # HTTP relay server + auth
│   └── bridges/                  # OpenClaw gateway bridges
├── agent_comm_mcp/               # MCP server (14 tools)
├── tests/                        # 212 pytest tests
├── scripts/                      # Integration tests (28 tests)
└── docs/                         # Documentation
```

---

## MessageV2 Protocol

All messages follow the MessageV2 schema:

```json
{
  "id": "msg_20260130_143022_a1b2c3",
  "version": "2.0",
  "from": "claude-code-pc1",
  "to": "nelly-pc2",
  "subject": "Research task",
  "body": "Research AI agent frameworks and summarize findings.",
  "timestamp": "2026-01-30T14:30:22Z",
  "priority": "high",
  "routing": {
    "reply_to": null,
    "conversation_id": "conv_001",
    "ttl_hours": 24
  },
  "payload": {
    "type": "task_assignment",
    "data": { "scope": "broad", "max_sources": 5 }
  }
}
```

### Priority Levels

| Priority | Description |
|----------|-------------|
| `urgent` | Immediate processing required |
| `high` | Process before normal messages |
| `normal` | Default priority |
| `low` | Process when queue is empty |

### Payload Types

| Type | Purpose |
|------|---------|
| `text` | Free-form message |
| `task_assignment` | Assign work to an agent |
| `task_result` | Report task completion |
| `context_share` | Share knowledge (reference Minna entities) |
| `status_request` / `status_response` | Agent status queries |
| `capability_query` / `capability_response` | Agent capability discovery |
| `broadcast` | Message to all registered agents |

### Delivery States

```
queued → leased → acked
                ↘ failed (after max_attempts)
```

| State | Description |
|-------|-------------|
| `queued` | Message waiting for recipient |
| `leased` | Claimed by recipient, processing |
| `acked` | Successfully processed |
| `failed` | Delivery failed after retries |

---

## Transport Layer

### Transport ABC

All transports implement the `Transport` abstract base class:

```python
class Transport(ABC):
    @abstractmethod
    def send(self, message: MessageV2) -> str:
        """Send message, return message_id"""

    @abstractmethod
    def poll(self, agent_id: str, limit: int = 10) -> list[MessageV2]:
        """Poll for messages addressed to agent"""

    @abstractmethod
    def ack(self, message_id: str, agent_id: str) -> bool:
        """Acknowledge message processing"""

    @abstractmethod
    def nack(self, message_id: str, agent_id: str, error: str = None) -> bool:
        """Negative acknowledge, return to queue"""
```

### SQLite Transport (Primary)

WAL-mode SQLite for local N-agent communication:

| Feature | Implementation |
|---------|---------------|
| **WAL mode** | Concurrent read/write |
| **Atomic claim** | `UPDATE WHERE status='queued' AND lease_until < NOW()` |
| **Idempotency** | `UNIQUE(message_id, recipient_agent_id)` |
| **Server-time TTL** | `expires_at` set using DB clock |

### HTTP Transport (Cross-device)

HMAC-SHA256 authenticated relay for PC-to-PC communication:

```
Agent A (PC1) → HTTP POST → Relay Server (PC2) → SQLite Spool → Agent B (PC2)
```

### File Transport (Debug)

JSON files in inbox directories for debugging:

```
data/inboxes/
├── agent-a/
│   └── msg_001.json
└── agent-b/
    └── msg_002.json
```

---

## Agent Registry

### Agent Profile

```python
@dataclass
class AgentProfile:
    agent_id: str
    capabilities: list[str]
    device: str
    transport: str = "sqlite"  # or "http"
    metadata: dict = field(default_factory=dict)
    status: str = "active"
    last_heartbeat: str = None
```

### Registry Operations

| Operation | Description |
|-----------|-------------|
| `register(profile)` | Add agent to registry |
| `discover(capability=None)` | Find agents by capability |
| `heartbeat(agent_id)` | Update last-seen timestamp |
| `deregister(agent_id)` | Remove from registry |

### Minna Memory Integration

Agent profiles can be stored in Minna Memory as entities:

```
Entity: agent:claude-code-pc1
Type: concept
Attributes:
  - capabilities: ["code", "mcp", "bash"]
  - device: "pc1"
  - transport: "sqlite"
```

---

## MCP Server

14 tools exposed via FastMCP:

### Messaging Tools

| Tool | Purpose | Hint |
|------|---------|------|
| `comm_send` | Send a message | |
| `comm_poll` | Poll for messages | readOnly |
| `comm_ack` | Acknowledge message | |
| `comm_nack` | Negative acknowledge | |

### Registry Tools

| Tool | Purpose | Hint |
|------|---------|------|
| `comm_register_agent` | Register agent | |
| `comm_discover_agents` | Find agents | readOnly |
| `comm_heartbeat` | Keep-alive | |
| `comm_deregister` | Remove agent | destructive |

### Status Tools

| Tool | Purpose | Hint |
|------|---------|------|
| `comm_message_status` | Delivery status | readOnly |
| `comm_spool_stats` | Queue statistics | readOnly |
| `comm_get_conversation` | Thread history | readOnly |

### Maintenance Tools

| Tool | Purpose | Hint |
|------|---------|------|
| `comm_cleanup` | Remove expired | destructive |
| `comm_relay_gen_secret` | Generate HMAC secret | |
| `comm_relay_list_secrets` | List secrets | readOnly |

---

## HTTP Relay

### Architecture

```
┌──────────────┐     HTTPS + HMAC      ┌──────────────┐
│   Agent A    │ ─────────────────────>│ Relay Server │
│   (PC1)      │                       │   (PC2)      │
│              │                       │              │
│ HTTPTransport│                       │  FastAPI +   │
│              │<─────────────────────│  SQLite Spool│
└──────────────┘     JSON Response     └──────────────┘
```

### HMAC-SHA256 Authentication

Each request includes:

| Header | Content |
|--------|---------|
| `X-Agent-ID` | Sender agent ID |
| `X-Timestamp` | ISO 8601 UTC timestamp |
| `X-Nonce` | Random 16-char hex |
| `X-Signature` | HMAC-SHA256(secret, timestamp + nonce + body) |

### Replay Protection

- 5-minute nonce tracking window
- Timestamp validation (±5 minutes)
- Per-agent nonce deduplication

### Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/v1/send` | POST | Send message |
| `/api/v1/poll/{agent_id}` | GET | Poll messages |
| `/api/v1/ack` | POST | Acknowledge |
| `/api/v1/nack` | POST | Negative ack |
| `/api/v1/status/{message_id}` | GET | Message status |
| `/api/v1/health` | GET | Health check |
| `/api/v1/agents/register` | POST | Register agent |

---

## Security Model

### Local Communication (SQLite)

- Filesystem-level access control
- WAL mode prevents corruption
- No authentication (trusted local agents)

### Cross-Device (HTTP Relay)

| Layer | Protection |
|-------|------------|
| **Transport** | HTTPS (TLS) |
| **Authentication** | HMAC-SHA256 per-agent secrets |
| **Replay** | Nonce + timestamp validation |
| **Authorization** | Agent can only poll own messages |

### Best Practices

1. **Keep relay secrets confidential** - Store in `data/relay-secrets.json`
2. **Use HTTPS in production** - Configure TLS certificates
3. **Run on trusted networks** - VPN, Tailscale, or LAN only
4. **Rotate secrets regularly** - Use `relay gen-secret` periodically

---

## Testing

### Test Coverage

| Category | Tests | Coverage |
|----------|-------|----------|
| Unit tests | 212 | All modules |
| Integration | 28 | End-to-end workflows |
| **Total** | **240** | |

### Running Tests

```bash
# All pytest tests
pytest tests/ -v

# Integration tests
python3 scripts/test_two_agents.py

# Everything
pytest tests/ -v && python3 scripts/test_two_agents.py
```

---

## Dependencies

| Package | Purpose |
|---------|---------|
| `pydantic>=2.0` | Message validation |
| `httpx>=0.25.0` | HTTP client for relay |
| `mcp[cli]>=1.0.0` | FastMCP server |
| `fastapi>=0.104.0` | Relay server (optional) |
| `uvicorn>=0.24.0` | ASGI server (optional) |

---

*Part of the [agentspool](https://github.com/fbratten/agentspool) project*
