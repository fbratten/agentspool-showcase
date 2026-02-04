---
layout: default
title: AI Agent Integration
nav_order: 2
description: "Complete guide for integrating agentspool with AI coding agents and OpenClaw gateway agents"
---

# AI Agent Integration Guide

**Purpose:** Enable bidirectional messaging between AI coding agents (running on Windows) and OpenClaw/Clawdbot gateway agents (running in WSL2)

**Version:** 1.0.0
**Status:** Production Ready (240 tests)

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Prerequisites](#prerequisites)
4. [Installation](#installation)
5. [Agent Registration](#agent-registration)
6. [Message Flow](#message-flow)
7. [Integration Patterns](#integration-patterns)
8. [End-to-End Walkthrough](#end-to-end-walkthrough)
9. [Troubleshooting](#troubleshooting)

---

## Overview

### The Problem

AI coding agents and OpenClaw gateway agents operate in different contexts:

| Component | Runtime | Location |
|-----------|---------|----------|
| AI Coding Agent | Windows terminal session | Windows 11 host |
| OpenClaw Agent | Gateway service | WSL2 Linux instance |

By default, communication is **one-way only**:

```
Coding Agent → OpenClaw Agent ✅  (via wsl -d <instance> -e openclaw agent --message "...")
OpenClaw Agent → Coding Agent ❌  (no mechanism)
```

### The Solution

**agentspool** provides a SQLite-backed message spool with:
- Atomic claim/lease semantics for reliable delivery
- Agent registry with capability discovery
- CLI tools for send/poll/ack workflows
- OpenClaw bridge for gateway integration
- MCP server with 14 tools for programmatic access

This enables **true bidirectional communication**:

```
Coding Agent ←→ Message Spool ←→ OpenClaw Agent
      ↓              ↓              ↓
    send()      coordination.db   poll()
    poll()                        send()
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Windows 11 Host                          │
│                                                                 │
│  ┌─────────────────┐                                           │
│  │  Coding Agent   │                                           │
│  │   (Terminal)    │                                           │
│  └────────┬────────┘                                           │
│           │                                                     │
│           │ wsl -d <instance> -e bash -c '...'                 │
│           ▼                                                     │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    WSL2 Instance                         │   │
│  │                                                          │   │
│  │  ┌──────────────┐    ┌──────────────┐    ┌───────────┐  │   │
│  │  │ agent-comm   │    │ Message      │    │ OpenClaw  │  │   │
│  │  │ CLI/Library  │◄──►│ Spool        │◄──►│ Gateway   │  │   │
│  │  └──────────────┘    │ (SQLite)     │    │ Agent     │  │   │
│  │                      └──────────────┘    └───────────┘  │   │
│  │                                                          │   │
│  │  Location: ~/projects/agentspool/data/coordination.db   │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### Key Components

| Component | Purpose | Location |
|-----------|---------|----------|
| `agent_comm` | Core Python library | `agent_comm/` |
| `coordination.db` | SQLite message spool | `data/` |
| `agent_comm_mcp` | MCP server (14 tools) | `agent_comm_mcp/` |
| CLI | Command-line interface | `python -m agent_comm <command>` |

---

## Prerequisites

### System Requirements

| Requirement | Minimum | Recommended |
|-------------|---------|-------------|
| Windows | Windows 10 21H2 | Windows 11 |
| WSL2 | Any distro | Ubuntu 24.04 |
| Python | 3.10+ | 3.12 |

### Software Dependencies

**On WSL2 instance:**
- Python 3.10+ with pip
- Git for cloning the repository

**On Windows host:**
- AI coding agent (CLI-based) installed
- WSL2 configured with target instance

---

## Installation

### Step 1: Clone the Repository

```bash
wsl -d <your-instance> -e bash -c '
mkdir -p ~/projects
cd ~/projects
git clone https://github.com/fbratten/agentspool.git
'
```

### Step 2: Create Virtual Environment

```bash
wsl -d <your-instance> -e bash -c '
cd ~/projects/agentspool
python3 -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"
'
```

### Step 3: Verify Installation

```bash
wsl -d <your-instance> -e bash -c '
cd ~/projects/agentspool
.venv/bin/python -m agent_comm --help
'
```

---

## Agent Registration

Before agents can communicate, they must be registered in the agent-comm registry.

### Register the OpenClaw Agent

```bash
.venv/bin/python -m agent_comm register my-assistant \
  -c "research,chat,gmail,tools" \
  -d wsl-main
```

### Register the Coding Agent

```bash
.venv/bin/python -m agent_comm register coding-agent \
  -c "code,implementation,bash,mcp" \
  -d windows-host
```

### Verify Registration

```bash
.venv/bin/python -m agent_comm agents
```

---

## Message Flow

### MessageV2 Protocol

All messages follow the MessageV2 schema:

```json
{
  "id": "msg_20260203_143022_a1b2c3",
  "version": "2.0",
  "from": "coding-agent",
  "to": "my-assistant",
  "subject": "Implementation task",
  "body": "Please implement the user authentication module.",
  "timestamp": "2026-02-03T14:30:22Z",
  "priority": "high",
  "routing": {
    "conversation_id": "conv_001",
    "ttl_hours": 24
  },
  "payload": {
    "type": "task_assignment",
    "data": { "scope": "feature" }
  }
}
```

### Delivery States

```
queued → leased → acked
                ↘ failed (after max_attempts)
```

### CLI Commands

#### Send a Message

```bash
.venv/bin/python -m agent_comm send coding-agent my-assistant \
  "Please research AI agent frameworks" \
  -s "Research task" \
  --priority high
```

#### Poll for Messages

```bash
.venv/bin/python -m agent_comm poll my-assistant
```

#### Acknowledge a Message

```bash
.venv/bin/python -m agent_comm ack msg_20260203_143022_a1b2c3 my-assistant
```

---

## Integration Patterns

### Pattern 1: Direct CLI (Simplest)

Execute agent-comm commands directly from your coding agent via bash:

```bash
# Send to gateway agent
wsl -d <instance> -e bash -c '
cd ~/projects/agentspool
.venv/bin/python -m agent_comm send coding-agent <agent> "Your message" -s "Subject"
'

# Poll for responses
wsl -d <instance> -e bash -c '
cd ~/projects/agentspool
.venv/bin/python -m agent_comm poll coding-agent
'
```

### Pattern 2: MCP Server Integration

agent-comm includes an MCP server with 14 tools:

| Tool | Purpose |
|------|---------|
| `comm_register_agent` | Register a new agent |
| `comm_send` | Send a message |
| `comm_poll` | Poll for pending messages |
| `comm_ack` | Acknowledge a message |
| `comm_nack` | Negative acknowledge |
| `comm_message_status` | Check delivery status |
| `comm_discover_agents` | List registered agents |
| `comm_spool_stats` | Spool statistics |
| `comm_cleanup` | Remove expired messages |
| `comm_get_conversation` | Get conversation history |
| `comm_heartbeat` | Update agent heartbeat |
| `comm_deregister` | Remove an agent |
| `comm_relay_gen_secret` | Generate relay HMAC secret |
| `comm_relay_list_secrets` | List relay secrets |

---

## End-to-End Walkthrough

### Step 1: Coding Agent Sends Task

```bash
wsl -d <instance> -e bash -c '
cd ~/projects/agentspool
.venv/bin/python -m agent_comm send coding-agent my-assistant \
  "Please analyze the authentication module" \
  -s "Code review request" \
  --priority high
'
```

### Step 2: Gateway Agent Polls and Processes

```bash
# Poll for messages
.venv/bin/python -m agent_comm poll my-assistant

# Forward to OpenClaw
openclaw agent --message "<message-body>" --json
```

### Step 3: Gateway Agent Sends Response

```bash
.venv/bin/python -m agent_comm send my-assistant coding-agent \
  "Analysis complete. Found 3 issues..." \
  -s "Re: Code review request"
```

### Step 4: Coding Agent Receives Response

```bash
.venv/bin/python -m agent_comm poll coding-agent
```

---

## Troubleshooting

### Common Issues

#### "Command not found: agent_comm"

**Fix:** Use full path to venv Python:
```bash
~/projects/agentspool/.venv/bin/python -m agent_comm --help
```

#### "No agents registered"

**Fix:** Register agents first:
```bash
.venv/bin/python -m agent_comm register <agent-name> -c "capability1,capability2" -d <device>
```

#### Message stuck in "leased" state

**Fix:** Nack to return to queue:
```bash
.venv/bin/python -m agent_comm nack <message-id> <agent>
```

### Diagnostic Commands

```bash
# View all messages and their states
.venv/bin/python -m agent_comm stats

# Check specific message status
.venv/bin/python -m agent_comm status <message-id>

# List all registered agents
.venv/bin/python -m agent_comm agents
```

---

## Environment Variables

| Variable | Purpose | Default |
|----------|---------|---------|
| `AGENT_COMM_DB_PATH` | SQLite database path | `./data/coordination.db` |
| `AGENT_COMM_PROJECT_ROOT` | Project root | repo root |
| `AGENT_COMM_TRANSPORT` | Transport backend | `sqlite` |
| `AGENT_COMM_TTL_HOURS` | Message TTL | `24` |
| `AGENT_COMM_LEASE_SECONDS` | Lease duration | `60` |

---

## CLI Quick Reference

| Command | Description |
|---------|-------------|
| `register <name>` | Register an agent |
| `send <from> <to> <body>` | Send a message |
| `poll <agent>` | Poll for messages |
| `ack <msg-id> <agent>` | Acknowledge message |
| `nack <msg-id> <agent>` | Negative acknowledge |
| `status <msg-id>` | Check delivery status |
| `agents` | List registered agents |
| `stats` | View spool statistics |
| `cleanup` | Remove expired messages |
| `bridge <agent>` | Run bridge daemon |
| `relay start` | Start HTTP relay server |

---

*Part of the [agentspool](https://github.com/fbratten/agentspool) project*
