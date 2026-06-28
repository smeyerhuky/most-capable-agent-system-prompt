---
type: Design Doc
title: "AgentForge System Architecture"
description: Full component diagram, data flow, and external integration map for the AgentForge cloud platform. Covers all components from the web IDE to the GPU fleet.
tags: [design, architecture, components, data-flow, aws, ecs, rds, redis]
timestamp: 2026-06-28T00:00:00Z
---

# AgentForge System Architecture

---

## Component Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  CLIENT TIER                                                                │
│                                                                             │
│  ┌──────────────┐  ┌────────────────┐  ┌─────────────────┐                │
│  │  Web IDE     │  │  VS Code Ext   │  │  CLI            │                │
│  │  (React SPA) │  │  (LSP + webview│  │  agentforge run │                │
│  └──────┬───────┘  └───────┬────────┘  └────────┬────────┘                │
│         └──────────────────┴───────────────────────┘                       │
│                             │ HTTPS + SSE                                  │
└─────────────────────────────┼───────────────────────────────────────────────┘
                              │
┌─────────────────────────────┼───────────────────────────────────────────────┐
│  CONTROL PLANE (AWS ECS Fargate, us-east-1)                                │
│                             │                                               │
│  ┌──────────────────────────▼──────────────────────────────────────────┐   │
│  │  API Gateway (CloudFront + ALB)                                     │   │
│  └──────────────────────────┬──────────────────────────────────────────┘   │
│              ┌──────────────┴───────────┐                                  │
│              │                          │                                   │
│  ┌───────────▼────────┐    ┌────────────▼───────┐                         │
│  │  API Server        │    │  Compiler Service  │                         │
│  │  (FastAPI/Python)  │    │  (stateless, scales│                         │
│  │  - REST endpoints  │    │   horizontally)    │                         │
│  │  - Auth/JWT        │    │  - Lexer           │                         │
│  │  - Billing hooks   │    │  - Parser          │                         │
│  └──────────┬─────────┘    │  - AST Builder     │                         │
│             │              │  - Symbol Table    │                         │
│             │              │  - Type Checker    │                         │
│  ┌──────────▼─────────┐    │  - IR Emitter      │                         │
│  │  Runtime Service   │    └────────────────────┘                         │
│  │  (async Python)    │                                                    │
│  │  - IR Walker       │                                                    │
│  │  - Tool Executor   │                                                    │
│  │  - Model Router    │                                                    │
│  │  - Event Emitter   │                                                    │
│  └──────┬─────────────┘                                                    │
│         │                                                                   │
└─────────┼───────────────────────────────────────────────────────────────────┘
          │
┌─────────┼───────────────────────────────────────────────────────────────────┐
│  DATA TIER                                                                  │
│         │                                                                   │
│  ┌──────▼──────────┐    ┌──────────────────┐    ┌──────────────────────┐  │
│  │  RDS Postgres   │    │  ElastiCache     │    │  S3                  │  │
│  │  (Multi-AZ)     │    │  Redis           │    │  - Pipeline sources  │  │
│  │  - users        │    │  - Run state     │    │  - IR files          │  │
│  │  - pipelines    │    │  - Event pub/sub │    │  - Run artifacts     │  │
│  │  - runs         │    │  - JWT cache     │    │  - Knowledge bundles │  │
│  │  - events       │    │  - Rate limits   │    └──────────────────────┘  │
│  │  - billing      │    └──────────────────┘                              │
│  │  - teams        │                                                       │
│  └─────────────────┘                                                       │
└─────────────────────────────────────────────────────────────────────────────┘
          │
┌─────────┼───────────────────────────────────────────────────────────────────┐
│  MODEL TIER                                                                 │
│         │                                                                   │
│  ┌──────▼──────────┐    ┌────────────────────┐    ┌──────────────────────┐│
│  │  AWS Bedrock    │    │  OpenAI API        │    │  Private GPU Fleet   ││
│  │  - Claude 3.5   │    │  - GPT-4o          │    │  (g5.xlarge spot)    ││
│  │  - Llama 3      │    │  - GPT-4o-mini     │    │  vLLM + Gemma 4      ││
│  │  - Titan        │    │  (User's own key)  │    │  26B-A4B + MTP       ││
│  └─────────────────┘    └────────────────────┘    └──────────────────────┘│
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Data Flow: Pipeline Execution

```
1. User submits POST /runs with pipeline_id

2. API Server:
   a. Authenticates JWT
   b. Checks execution quota (Redis rate limit key)
   c. Fetches pipeline source from S3
   d. POSTs to Compiler Service → receives IR
   e. Creates run record in Postgres (status: 'running')
   f. Publishes run_id to Runtime Service queue (Redis pub/sub)
   g. Returns { run_id, status: 'running' }

3. Runtime Service receives run_id:
   a. Fetches IR from S3
   b. Initializes ExecutionState in Redis
   c. Begins IR walk loop

4. For each instruction:
   a. LLM_CALL → Model Router → Bedrock/OpenAI/vLLM API
   b. CALL → Tool Executor → filesystem/subprocess/MCP
   c. CHECKPOINT → Emits CheckpointPaused event → suspends
   d. All instructions emit events to Redis pub/sub

5. SSE endpoint (GET /runs/{id}/stream):
   a. API Server subscribes to Redis pub/sub channel for run_id
   b. Relays events to client as SSE

6. On run completion:
   a. Runtime writes final state to Postgres
   b. Billing service reads token counts and posts to Stripe
   c. Emits RunCompleted event
```

---

## External Integrations

| Service | Purpose | Auth Method |
|---|---|---|
| AWS Bedrock | Claude and Llama 3 inference (Tier 1) | IAM Role (ECS task role) |
| OpenAI API | GPT-4o inference (user's own key) | Per-user API key (encrypted in Postgres) |
| AWS KMS | API key encryption | IAM Role |
| Stripe | Subscription billing + metered usage | Stripe Secret Key (environment variable) |
| AWS Cost Explorer | Bedrock cost reconciliation | IAM Role |
| GitHub (MCP) | GitHub operations in pipelines | Per-user PAT (encrypted) |
| Clerk / Auth0 | User authentication | OAuth 2.0 |
| CloudWatch | Metrics, logs, alerts | IAM Role |
| PagerDuty | On-call alerting (Phase 3) | PagerDuty API Key |

---

## Phase 1 Topology

Phase 1 uses a simplified deployment to minimize operational complexity:

- Single ECS Fargate cluster in us-east-1
- API Server + Compiler Service + Runtime Service co-deployed (separate containers, same cluster)
- Single RDS Postgres instance (Multi-AZ enabled from day one for reliability)
- Single ElastiCache Redis node
- CloudFront CDN for web IDE static assets (S3 origin)

Phase 3 adds multi-region and the GPU fleet — see [gpu-serving-design.md](gpu-serving-design.md).

---

## Related

- [Compiler Design](compiler-design.md) — Compiler Service internals
- [Runtime Design](runtime-design.md) — Runtime Service internals
- [Data Model](data-model.md) — Postgres schema
- [GPU Serving Design](gpu-serving-design.md) — Private GPU Fleet
- [Billing Design](billing-design.md) — Stripe integration
