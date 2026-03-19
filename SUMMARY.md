# AgentField — Summary

AgentField is an open-source control plane that turns AI agent functions into production-grade REST endpoints. You write agent logic in Python, Go, or TypeScript; AgentField handles routing, coordination, memory, async execution, canary deployments, and cryptographic audit trails. Every agent function becomes a REST endpoint, every agent gets a cryptographic identity (W3C DID), and every execution produces a tamper-proof audit trail.

---

## Table of Contents

1. [What It Does](#what-it-does)
2. [Architecture](#architecture)
3. [Features](#features)
4. [Installation](#installation)
5. [Usage](#usage)
   - [Python SDK](#python-sdk)
   - [Go SDK](#go-sdk)
   - [REST API](#rest-api)
   - [CLI Reference](#cli-reference)
6. [Configuration](#configuration)
7. [Further Reading](#further-reading)

---

## What It Does

Modern AI agents go beyond simple chat. They approve refunds, process insurance claims, orchestrate research pipelines, and run code autonomously — all inside backend systems. Those workloads need the same infrastructure that traditional microservices already rely on:

| Traditional Backend | AgentField for AI Agents |
|---|---|
| REST endpoint registration | `@app.reasoner()` auto-exposes as `POST /api/v1/execute/…` |
| Service discovery | `app.discover(tags=["billing*"])` — tag-based mesh lookup |
| Async workers + queues | Fire-and-forget with webhooks and SSE streaming |
| Distributed state / caching | `app.memory.set/get/search` — KV + vector, four scopes, no Redis |
| IAM / identity | W3C DID per agent, Ed25519 signatures, Verifiable Credentials |
| Canary / blue-green deploys | Traffic-weight routing at the control-plane level |
| Observability | Automatic DAG visualisation, Prometheus metrics, structured logs |

AgentField is a **control plane**, not an agent framework. It sits between agents and your stack, routing calls, tracking workflows, and enforcing policies.

---

## Architecture

AgentField is a three-tier monorepo:

```
agentfield/
├── control-plane/          # Go orchestration server (REST + gRPC)
├── sdk/
│   ├── python/             # Python SDK (FastAPI/Uvicorn agents)
│   └── go/                 # Go SDK (idiomatic agent builder)
└── control-plane/web/      # React/TypeScript dashboard (embedded in binary)
```

### Control Plane

The control plane is a stateless Go service built on [Gin](https://github.com/gin-gonic/gin). It:

- Receives agent registrations and maintains the agent registry
- Routes synchronous and asynchronous execution requests
- Tracks every call as a node in a directed acyclic graph (DAG) — the *workflow*
- Issues and verifies W3C Verifiable Credentials for every execution
- Persists state in either **local mode** (SQLite + BoltDB, zero external dependencies) or **cloud mode** (PostgreSQL)

```
Clients (SDK / Web UI / REST)
         │
         ▼
┌─────────────────────────────┐
│        Control Plane        │
│  handlers → services        │
│  storage (SQLite / Postgres)│
│  events (SSE / webhooks)    │
│  encryption (DID / VC)      │
└─────────────────────────────┘
         │
         ▼
   Registered Agents
 (Python / Go / TypeScript)
```

**Internal packages** (`control-plane/internal/`):

| Package | Purpose |
|---|---|
| `handlers/` | HTTP request handlers |
| `services/` | Business logic (workflows, registry, DID/VC) |
| `storage/` | Data persistence (SQLite / BoltDB / PostgreSQL) |
| `events/` | Event bus — SSE streaming and workflow notifications |
| `encryption/` | Ed25519 keys, W3C DID, Verifiable Credentials |
| `mcp/` | Model Context Protocol integration |
| `config/` | Viper-based configuration |
| `logger/` | Structured logging (zerolog) |

### SDKs

Agents are HTTP servers that register themselves with the control plane on startup.

- **Python SDK** — FastAPI/Uvicorn agent server. Decorators (`@app.reasoner`, `@app.skill`) expose functions as REST endpoints and auto-register with the control plane.
- **Go SDK** — Idiomatic Go agent builder (`agent.RegisterReasoner`, `agent.RegisterSkill`).

### Web Dashboard

The React/TypeScript dashboard is embedded in the server binary and served at `http://localhost:8080/ui/`. It provides:

- Real-time workflow DAG visualisation
- Execution traces and timelines
- Agent fleet management
- Human-in-the-loop approval UI

---

## Features

### Build

| Capability | How |
|---|---|
| Auto-REST from decorators | `@app.reasoner()` → `POST /api/v1/execute/{agent}.{func}` |
| Structured LLM output | `app.ai(schema=MyPydanticModel)` — typed output from any LLM |
| 100+ LLM providers | `AIConfig(model="anthropic/claude-sonnet-4-…")` via LiteLLM |
| Multi-turn coding agents | `app.harness("Fix the bug", provider="claude-code")` |
| Cross-agent calls | `app.call("other-agent.func", input={…})` — routed with full tracing |
| Tag-based agent discovery | `app.discover(tags=["ml*"])` — wildcard support |
| LLM auto-discovers tools | `app.ai(tools="discover")` — agents appear as LLM tool calls |
| Distributed memory | `app.memory.set/get/search` — KV + vector, four scopes |
| Parallel execution | `asyncio.gather(app.call(…), app.call(…))` |

### Run

| Capability | How |
|---|---|
| Sync REST execution | `POST /api/v1/execute/{agent}.{func}` |
| Async fire-and-forget | `POST /api/v1/execute/async/{agent}.{func}` |
| SSE streaming | `GET /api/v1/execute/stream/{id}` |
| Webhooks + HMAC-SHA256 | `AsyncConfig(webhook_url="…", secret="…")` |
| No timeout limits | Agents run for hours or days |
| Human-in-the-loop | `await app.pause(…)` — suspends execution, crash-safe, durable |
| Auto retries | Exponential backoff, transparent to callers |
| Canary deployments | Traffic-weight routing: 5 % → 50 % → 100 % |
| A/B testing | 50 / 50 splits, `X-Routed-Version` header |
| Blue-green deploys | Instant weight switch, zero downtime |

### Govern

| Capability | How |
|---|---|
| Cryptographic agent identity | Auto-generated W3C DID + Ed25519 keypairs per agent |
| Verifiable Credentials | Tamper-proof execution receipt, offline-verifiable |
| VC verification | `af vc verify audit.json` |
| Tag-based policy gates | ALLOW / DENY rules enforced by infrastructure, not prompts |
| Signed cross-agent requests | Ed25519 signatures on every inter-agent call |
| Audit notes | `app.note("Decision", tags=["critical"])` |

### Observe

| Capability | How |
|---|---|
| Workflow DAG | Auto-built; visualised in the dashboard |
| Prometheus metrics | `GET /metrics` — out of the box |
| Structured logs | JSON from the SDK and control plane |
| Execution timeline | Chronological decision trace per workflow |
| Health probes | `GET /health`, `GET /ready` — K8s-ready |
| Correlation IDs | `X-Workflow-ID`, `X-Execution-ID` headers |

---

## Installation

### Prerequisites

- Go ≥ 1.23
- Python ≥ 3.8
- Node.js ≥ 20
- PostgreSQL ≥ 15 (optional — only needed for cloud/production mode)

### CLI

```bash
curl -fsSL https://agentfield.ai/install.sh | bash
```

### From Source

```bash
git clone https://github.com/Agent-Field/agentfield.git
cd agentfield
make install   # installs Go tooling, Python SDK, and Node packages
make build     # compiles control plane + embeds web UI
```

### Docker

```bash
# Control plane only
docker run -p 8080:8080 agentfield/control-plane:latest

# Full local stack (control plane + PostgreSQL)
cd deployments/docker
docker compose up
```

---

## Usage

### Quick Start

```bash
# 1. Scaffold a new agent
af init my-agent --defaults
cd my-agent && pip install -r requirements.txt

# 2. Start the control plane (Terminal 1)
af server
# Dashboard at http://localhost:8080/ui/

# 3. Start the agent (Terminal 2)
python main.py
# Agent auto-registers with the control plane

# 4. Call the agent
curl -X POST http://localhost:8080/api/v1/execute/my-agent.demo_echo \
  -H "Content-Type: application/json" \
  -d '{"input": {"message": "Hello!"}}'
```

---

### Python SDK

#### Installation

```bash
pip install agentfield
```

#### Defining an Agent

```python
from agentfield import Agent, AIConfig
from pydantic import BaseModel

app = Agent(
    node_id="claims-processor",
    version="2.1.0",
    ai_config=AIConfig(model="anthropic/claude-sonnet-4-20250514"),
)
```

#### Reasoners and Skills

```python
# Reasoner — AI-powered judgment
@app.reasoner(tags=["insurance", "critical"])
async def evaluate_claim(claim: dict) -> dict:
    ...

# Skill — deterministic function
@app.skill()
async def format_report(data: dict) -> dict:
    ...
```

#### Structured AI Output

```python
class Decision(BaseModel):
    action: str        # "approve", "deny", "escalate"
    confidence: float
    reasoning: str

@app.reasoner()
async def evaluate(claim: dict) -> dict:
    decision = await app.ai(
        system="Insurance claims adjuster.",
        user=f"Claim: {claim['description']}",
        schema=Decision,
    )
    return decision.model_dump()
```

#### Human-in-the-Loop

```python
@app.reasoner()
async def review_order(order: dict) -> dict:
    if order["amount"] > 10_000:
        await app.pause(
            approval_request_id=f"order-{order['id']}",
            approval_request_url=f"https://example.com/approvals/{order['id']}",
            expires_in_hours=48,
        )
    return {"status": "approved"}
```

#### Cross-Agent Calls

```python
@app.reasoner()
async def process(data: dict) -> dict:
    result = await app.call("analytics.summarize", input={"data": data})
    return result
```

#### Memory

```python
# Store state (agent scope)
await app.memory.set("last_run", {"ts": "2025-01-01"}, scope="agent")

# Retrieve
value = await app.memory.get("last_run", scope="agent")

# Vector search
results = await app.memory.search(embedding=[0.1, 0.2, …], top_k=5)
```

#### Running the Agent

```python
if __name__ == "__main__":
    app.run()   # Starts Uvicorn; registers with control plane
```

---

### Go SDK

#### Installation

```bash
go get github.com/Agent-Field/agentfield/sdk/go
```

#### Defining an Agent

```go
package main

import (
    "context"
    "fmt"

    agentfieldagent "github.com/Agent-Field/agentfield/sdk/go/agent"
)

func main() {
    ag, _ := agentfieldagent.New(agentfieldagent.Config{
        NodeID:        "greeter",
        AgentFieldURL: "http://localhost:8080",
        ListenAddress: ":8001",
        PublicURL:     "http://localhost:8001",
    })

    ag.RegisterReasoner("say_hello",
        func(ctx context.Context, input map[string]any) (any, error) {
            name, _ := input["name"].(string)
            if name == "" {
                name = "World"
            }
            return map[string]any{"greeting": fmt.Sprintf("Hello, %s!", name)}, nil
        },
        ag.WithDescription("Greets a person by name"),
    )

    ag.Run(context.Background())
}
```

#### Scaffolding

```bash
af init my-agent --defaults --language go
cd my-agent && go run .
```

---

### REST API

All endpoints are relative to the control plane base URL (default `http://localhost:8080`).

#### Execution

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/v1/execute/{agent}.{func}` | Synchronous execution |
| `POST` | `/api/v1/execute/async/{agent}.{func}` | Async fire-and-forget |
| `GET` | `/api/v1/execute/stream/{id}` | SSE streaming |
| `GET` | `/api/v1/executions/{id}` | Poll execution status |
| `POST` | `/api/v1/executions/batch-status` | Poll multiple executions |

#### Agents

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/agents` | List all registered agents |
| `GET` | `/api/v1/agents/{id}/details` | Agent details and reasoners |
| `GET` | `/api/v1/agents/{id}/status` | Agent health status |
| `POST` | `/api/v1/agents/{id}/start` | Start agent |
| `POST` | `/api/v1/agents/{id}/stop` | Stop agent |

#### Workflows

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/workflows/{id}/dag` | Workflow DAG |
| `GET` | `/api/v1/workflows/{id}/vc-chain` | VC audit chain for a workflow |

#### Memory

| Method | Path | Description |
|---|---|---|
| `GET` / `POST` | `/api/v1/memory/{scope}` | Key-value operations |
| `POST` | `/api/v1/memory/search` | Semantic vector search |

#### Identity (DID / VC)

| Method | Path | Description |
|---|---|---|
| `GET` | `/.well-known/did.json` | Control plane DID document |
| `GET` | `/agents/{id}/did.json` | Agent DID document |
| `GET` | `/api/v1/executions/{id}/vc` | Execution Verifiable Credential |
| `POST` | `/api/v1/executions/{id}/verify-vc` | Verify a VC offline |

#### Observability

| Method | Path | Description |
|---|---|---|
| `GET` | `/metrics` | Prometheus metrics |
| `GET` | `/health` | Liveness probe |
| `GET` | `/ready` | Readiness probe |

#### Execution Request / Response

```jsonc
// POST /api/v1/execute/claims-processor.evaluate_claim
{
  "input": { "id": "CLM-001", "description": "Burst pipe, $4500 repair" },
  "workflow_id": "wf-abc123",   // optional — links to existing workflow
  "actor_id": "user-42"         // optional — propagated to all downstream calls
}

// Response (200 OK)
{
  "execution_id": "exec-xyz789",
  "status": "completed",
  "result": { "action": "approve", "confidence": 0.94, "reasoning": "…" },
  "workflow_id": "wf-abc123",
  "duration_ms": 1243
}
```

---

### CLI Reference

| Command | Description |
|---|---|
| `af server` | Start the AgentField control plane |
| `af init <name>` | Scaffold a new agent (Python / Go / TypeScript) |
| `af run` | Run an agent locally and connect it to the control plane |
| `af dev` | Development mode with hot reload |
| `af list` | List all registered agents |
| `af execute <agent>.<func>` | Invoke an agent function via the CLI |
| `af logs <agent-id>` | Stream agent logs |
| `af stop <agent-id>` | Stop a running agent |
| `af vc verify <file>` | Verify a Verifiable Credential audit file offline |
| `af add --mcp --url <url>` | Add an MCP server to an agent |
| `af config` | Manage control plane configuration |
| `af --version` | Print version information |

**Common flags for `af server`:**

```bash
af server --port 9000           # Custom port (default: 8080)
af server --storage postgresql  # Use PostgreSQL backend
af server --ui-dev              # Proxy UI to local Vite dev server
```

**Scaffolding options for `af init`:**

```bash
af init my-agent --language python     # Python (default)
af init my-agent --language go
af init my-agent --language typescript
af init my-agent --defaults            # Accept all defaults, no prompts
```

---

## Configuration

Configuration is loaded from (highest to lowest priority):

1. Environment variables
2. `config/agentfield.yaml` (or the path in `AGENTFIELD_CONFIG_FILE`)
3. Built-in defaults

### Key Environment Variables

| Variable | Default | Description |
|---|---|---|
| `AGENTFIELD_PORT` | `8080` | HTTP server port |
| `AGENTFIELD_MODE` | `local` | `local` or `cloud` |
| `AGENTFIELD_STORAGE_MODE` | `local` | `local`, `postgresql`, or `cloud` |
| `AGENTFIELD_DATABASE_URL` | — | PostgreSQL DSN (cloud/postgresql mode) |
| `AGENTFIELD_HOME` | `~/.agentfield` | Base directory for local state |
| `AGENTFIELD_API_KEY` | — | API key for authentication |
| `AGENTFIELD_AUTHORIZATION_ENABLED` | `false` | Enable VC-based access control |
| `AGENTFIELD_CONNECTOR_TOKEN` | — | Bearer token for the Connector API |
| `GIN_MODE` | `debug` | `debug` or `release` |
| `LOG_LEVEL` | `info` | `debug`, `info`, `warn`, or `error` |

See [`docs/ENVIRONMENT_VARIABLES.md`](docs/ENVIRONMENT_VARIABLES.md) for the complete reference.

### Storage Modes

**Local** (default — no external dependencies):

```yaml
storage:
  mode: "local"
  local:
    database_path: "~/.agentfield/data/agentfield.db"
    kv_store_path: "~/.agentfield/data/kv.db"
```

**PostgreSQL**:

```yaml
storage:
  mode: "postgresql"
  postgres:
    host: "localhost"
    port: 5432
    database: "agentfield"
    user: "agentfield"
    password: "agentfield"
    ssl_mode: "disable"
```

Run migrations before starting the server in PostgreSQL mode:

```bash
goose -dir control-plane/migrations postgres "$AGENTFIELD_DATABASE_URL" up
```

---

## Further Reading

| Document | Description |
|---|---|
| [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) | High-level architecture overview |
| [`docs/DEVELOPMENT.md`](docs/DEVELOPMENT.md) | Development setup and testing |
| [`docs/ENVIRONMENT_VARIABLES.md`](docs/ENVIRONMENT_VARIABLES.md) | Complete environment variable reference |
| [`docs/VC_AUTHORIZATION_ARCHITECTURE.md`](docs/VC_AUTHORIZATION_ARCHITECTURE.md) | DID and Verifiable Credential deep-dive |
| [`docs/CONTRIBUTING.md`](docs/CONTRIBUTING.md) | Contribution guidelines |
| [`docs/RELEASE.md`](docs/RELEASE.md) | Release process |
| [Full Documentation](https://agentfield.ai/docs) | Official documentation site |
| [Examples](https://agentfield.ai/examples) | Example projects built with AgentField |
