---
type: PRD Phase
title: "Phase 3 — Scale"
description: Scale phase requirements and agentic PDLC task decomposition for AgentForge. Covers multi-region Gemma 4 GPU deployment, enterprise tier, team workspaces, and AWS Marketplace listing. Target: $10,000 MRR, 2 enterprise pilots.
tags: [prd, scale, phase-3, tasks, gpu, enterprise, aws-marketplace, team]
timestamp: 2026-06-28T00:00:00Z
---

# Phase 3 — Scale

**Duration:** Months 5–6 (8 weeks)
**Gate:** [Phase 3 Go/No-Go](../04-validation/go-no-go-criteria.md#phase-3-gate)
**Goal:** Enterprise-ready platform with private GPU tier, team workspaces, AWS Marketplace listing, and $10,000 MRR.

---

## What Ships in Phase 3

1. Private GPU tier: Gemma 4-26B-A4B + MTP on AWS `g5.xlarge` spot instances (single region initially)
2. Team workspaces: shared pipelines, member management, team billing
3. AWS Marketplace listing (SaaS subscription via AWS billing)
4. Enterprise auth: SSO via SAML 2.0 (Okta, Azure AD)
5. Audit logs: full run history exportable for compliance
6. LangGraph pipeline export
7. Multi-region control plane (us-east-1, eu-west-1)
8. SLA monitoring + status page

---

## Task Decomposition

### Milestone 3A: Private GPU Tier (Week 17–20)

**TASK-3-01: vLLM Serving Stack**
- Owner: agent (infra configuration) + founder (AWS account + GPU quota request)
- Acceptance criteria: vLLM deployed on AWS `g5.xlarge` spot instance with auto-restart on preemption. Serves `google/gemma-4-26B-A4B-it` with AWQ quantization and speculative decoding via `google/gemma-4-26B-A4B-it-assistant`. OpenAI-compatible API at `POST /v1/chat/completions`. Health check endpoint. Inference latency P50 < 300ms first token.
- Depends on: TASK-1-09
- Effort: XL

**TASK-3-02: Gemma 4 Model Adapter**
- Owner: agent
- Acceptance criteria: `LLM_CALL` with `model=gemma4-26b-private` routes to the vLLM endpoint. Falls back to Bedrock Llama 3 if vLLM health check fails. Token tracking. MTP acceptance rate logged per call. Adapter plugs into the model routing system identically to the Bedrock adapter.
- Depends on: TASK-3-01, TASK-1-09
- Effort: S

**TASK-3-03: Spot Instance Failover**
- Owner: agent
- Acceptance criteria: Health check polls vLLM endpoint every 30s. On failure: pauses queued LLM_CALL instructions, attempts restart, falls back to Bedrock after 60s of downtime. Execution resumes from the paused instruction when GPU recovers. Users see a "GPU tier temporarily unavailable — using Bedrock fallback" notice in the dashboard.
- Depends on: TASK-3-02
- Effort: M

**TASK-3-04: GPU Billing Metering**
- Owner: agent
- Acceptance criteria: GPU tier token usage tracked separately from Bedrock usage. GPU tier included in Builder/Team plans up to a monthly credit limit (defined in [revenue-model.md](../07-business-plan/revenue-model.md)). Overage billed at $0.10/1M tokens. Stripe metered billing integration.
- Depends on: TASK-3-02, TASK-1-18
- Effort: M

**TASK-3-05: Dashboard GPU Metrics**
- Owner: agent
- Acceptance criteria: Dashboard shows per-run: GPU tokens/sec, MTP acceptance rate, time-to-first-token, cost comparison (GPU tier vs. Bedrock equivalent). "Why is the GPU tier faster?" tooltip with MTP explanation. Historical GPU usage chart on account settings page.
- Depends on: TASK-3-04, TASK-1-16
- Effort: M

### Milestone 3B: Team Workspaces (Week 19–21)

**TASK-3-06: Team Data Model**
- Owner: agent
- Acceptance criteria: Schema additions per [data-model.md](../06-design/data-model.md): `teams`, `team_members`, `team_pipelines` tables. Pipeline ownership: `user_id` OR `team_id`. Member roles: `owner`, `admin`, `member`. All existing user queries remain valid (no breaking migration).
- Depends on: TASK-1-19
- Effort: M

**TASK-3-07: Team Management UI**
- Owner: agent
- Acceptance criteria: Settings page: "Create Team", invite by email, assign roles, remove members. Shared pipeline library: all team members can view, fork, and run team pipelines. Pipeline visibility toggle: private (owner only), team, public. Billing: team plan billed to team owner's Stripe customer.
- Depends on: TASK-3-06, TASK-1-17
- Effort: L

**TASK-3-08: Team Billing (Seat-Based)**
- Owner: agent
- Acceptance criteria: Team plan: $99/seat/month. Adding a seat immediately prorates the Stripe subscription. Removing a seat credits the next invoice. Seat limit enforced at invite time. Team owner receives consolidated invoice.
- Depends on: TASK-3-07, TASK-1-18
- Effort: M

### Milestone 3C: Enterprise Auth (Week 20–22)

**TASK-3-09: SAML 2.0 SSO**
- Owner: agent (SAML library integration) + founder (Okta/Azure AD configuration for pilot customers)
- Acceptance criteria: SAML SP-initiated SSO flow. IdP metadata URL import. Attribute mapping: email, name, groups (mapped to team roles). Session via existing JWT auth. Works with Okta, Azure AD, and Google Workspace as IdPs. SSO-only mode: disable email/password login for a team (enterprise requirement).
- Depends on: TASK-3-07, TASK-1-17
- Effort: L

**TASK-3-10: Audit Logs**
- Owner: agent
- Acceptance criteria: All runs, compile events, user actions (invite, remove, role change, pipeline create/delete/run) logged with: timestamp, user_id, team_id, action, resource_id, IP address. `GET /audit-logs` API (paginated, filterable by date/user/action). CSV export. 90-day retention. Logs tamper-evident (append-only table with checksum chain).
- Depends on: TASK-3-06
- Effort: M

### Milestone 3D: AWS Marketplace (Week 21–24)

**TASK-3-11: AWS Marketplace SaaS Listing**
- Owner: founder (AWS Partner Network account + listing submission)
- Acceptance criteria: SaaS subscription listing on AWS Marketplace. Supports all three paid tiers (Builder, Team, Enterprise). AWS Marketplace Metering Service integrated for usage tracking. Customers can subscribe using their existing AWS account (no new payment relationship). Listing approved by AWS.
- Depends on: TASK-1-18, TASK-3-08
- Effort: L

**TASK-3-12: AWS ISV Accelerate Enrollment**
- Owner: founder
- Acceptance criteria: AWS ISV Accelerate program application submitted. Co-sell materials prepared (1-page solution brief, customer reference story). AWS Partner Solutions Architect introduction call scheduled.
- Depends on: TASK-3-11
- Effort: M

### Milestone 3E: LangGraph Export (Week 22–23)

**TASK-3-13: LangGraph Pipeline Exporter**
- Owner: agent
- Acceptance criteria: `agentforge export --format langgraph <file.agent>` emits a Python file containing a valid LangGraph `StateGraph`. All step nodes map to LangGraph nodes. Branch edges map to conditional edges. Loop nodes map to LangGraph cycles. Export tested against LangGraph 0.2+ API. Exported graph runs correctly in a fresh LangGraph environment.
- Depends on: TASK-1-06
- Effort: L

### Milestone 3F: Multi-Region + SLA (Week 23–24)

**TASK-3-14: Multi-Region Control Plane**
- Owner: agent + founder (AWS us-east-1 + eu-west-1 setup)
- Acceptance criteria: REST API and web IDE deployed in two regions. Route 53 latency-based routing. Postgres read replicas in EU. Run execution is region-local (data residency). User can select preferred region in account settings.
- Depends on: TASK-1-13
- Effort: XL

**TASK-3-15: Status Page + SLA Monitoring**
- Owner: agent
- Acceptance criteria: Public status page (statuspage.io or self-hosted Upptime) showing uptime for: web IDE, API, Bedrock routing, GPU tier. Alerts: PagerDuty or email on P1 incidents. Monthly uptime report. Enterprise SLA: 99.9% uptime commitment for Enterprise tier, measured and reported monthly.
- Depends on: TASK-3-14
- Effort: M

---

## Phase 3 Exit Criteria

- ≥ $10,000 MRR
- Private GPU tier running and handling ≥ 20% of paid-tier LLM calls
- ≥ 5 team workspace accounts
- ≥ 2 enterprise pilot customers (on SSO + audit logs)
- AWS Marketplace listing live and accepting subscriptions
- Multi-region control plane in us-east-1 and eu-west-1
- Status page with public uptime history

---

## Related

- [Phase 2 Growth](phase-2-growth.md) — prerequisite
- [GPU Serving Design](../06-design/gpu-serving-design.md) — architecture for TASK-3-01
- [Data Model](../06-design/data-model.md) — schema additions for TASK-3-06
- [Revenue Model](../07-business-plan/revenue-model.md) — pricing definitions for billing tasks
- [Go-to-Market](../07-business-plan/go-to-market.md) — AWS Marketplace strategy
