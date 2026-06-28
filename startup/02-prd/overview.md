---
type: PRD Overview
title: "AgentForge PRD Overview"
description: Design principles, three-phase delivery map, and success metrics for AgentForge. This document is the anchor for all phase-level requirements.
tags: [prd, overview, principles, metrics, phases]
timestamp: 2026-06-28T00:00:00Z
---

# AgentForge PRD Overview

## Design Principles

These principles govern every product and engineering decision. When tradeoffs arise, resolve them in priority order.

1. **Zero friction to first run.** A developer must be able to write their first `.agent` file and see it execute in production in under 15 minutes. Every feature that adds setup steps must justify itself against this constraint.

2. **Compile-time over runtime errors.** A type error caught by the compiler is worth 10x a runtime failure. The compiler is a feature, not infrastructure. Invest in error message quality.

3. **The language is the product.** The runtime, the dashboard, and the billing are implementation details. The `.agent` DSL is the durable value — design it with the care of a public API.

4. **Portability is the moat.** Every feature that locks a workflow to one model provider is a feature we are specifically not building. The model is a parameter, not a dependency.

5. **Observable by default.** Every step, token, tool call, and model response is logged and queryable. The dashboard is not a nice-to-have — it is the primary debugging surface.

6. **Product-led growth.** Every feature should enable a user to share something that makes their peers want to try AgentForge. The VS Code extension, GitHub Actions integration, and public pipeline templates are all PLG vectors.

---

## Phase Map

| Phase | Name | Months | Goal |
|---|---|---|---|
| 1 | MVP | 1–2 | Paying users running `.agent` files in production on Bedrock |
| 2 | Growth | 3–4 | Developer ecosystem adoption via VS Code, MCP, GitHub Actions |
| 3 | Scale | 5–6 | Enterprise-ready: private GPU tier, team workspaces, AWS Marketplace |

---

## Success Metrics Per Phase

### Phase 1 — MVP

| Metric | Target | Measurement |
|---|---|---|
| Time to first run (new user) | < 15 minutes | Onboarding funnel analytics |
| Paying users | ≥ 10 at $29/month | Stripe |
| Compile error rate | < 5% of valid programs rejected | Compiler telemetry |
| Runtime uptime | ≥ 99.0% | AWS CloudWatch |
| Bedrock pass-through billing accuracy | 100% | Reconciliation script |

### Phase 2 — Growth

| Metric | Target | Measurement |
|---|---|---|
| GitHub stars | ≥ 500 | GitHub |
| VS Code extension installs | ≥ 1,000 | VS Code Marketplace |
| HN Show HN upvotes | ≥ 200 | Hacker News |
| Monthly active pipelines | ≥ 100 | Runtime telemetry |
| Pipeline templates in marketplace | ≥ 10 | Marketplace database |

### Phase 3 — Scale

| Metric | Target | Measurement |
|---|---|---|
| MRR | ≥ $10,000 | Stripe |
| Private GPU tier users | ≥ 20 | Billing system |
| Team workspace accounts | ≥ 5 | Auth system |
| AWS Marketplace listing live | Yes | AWS Partner Network |
| Enterprise pilot customers | ≥ 2 | CRM |

---

## Agentic PDLC Task Conventions

Each phase document decomposes work into tasks following this schema:

```
TASK-{phase}-{number}: {Title}
  Owner: agent | founder | contractor
  Acceptance criteria: what "done" means in observable terms
  Depends on: TASK-{n}
  Estimated effort: S / M / L / XL
```

- **Owner: agent** — an LLM coding agent can complete this task given the spec and design docs as context
- **Owner: founder** — requires founder judgment, external relationships, or account access
- **Owner: contractor** — specialized work that can be outsourced (design, legal, copywriting)

Tasks are ordered by dependency graph. Parallel tasks (no dependency on each other) can be dispatched to agents concurrently.

---

## Related

- [Phase 1 MVP](phase-1-mvp.md)
- [Phase 2 Growth](phase-2-growth.md)
- [Phase 3 Scale](phase-3-scale.md)
- [Go/No-Go Criteria](../04-validation/go-no-go-criteria.md)
