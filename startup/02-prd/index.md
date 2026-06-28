---
type: Index
title: "PRD Index"
description: Navigation index for the AgentForge Product Requirements Document. Three delivery phases with agentic PDLC task decomposition.
tags: [prd, index, phases, okf]
timestamp: 2026-06-28T00:00:00Z
---

# Product Requirements Document

The PRD is organized into an overview and three delivery phases. Each phase document contains agentic PDLC task decomposition — tasks small enough for an LLM coding agent to execute autonomously, with explicit acceptance criteria and inter-task dependencies.

## Documents

| Document | Description |
|---|---|
| [overview.md](overview.md) | Design principles, phase map, success metrics |
| [phase-1-mvp.md](phase-1-mvp.md) | MVP: harness core + Bedrock + web IDE (Months 1–2) |
| [phase-2-growth.md](phase-2-growth.md) | Growth: VS Code ext, MCP, GitHub Actions, marketplace (Months 3–4) |
| [phase-3-scale.md](phase-3-scale.md) | Scale: multi-region GPU, enterprise tier, AWS Marketplace (Months 5–6) |

## Phase Gates

Each phase has an explicit go/no-go gate defined in [04-validation/go-no-go-criteria.md](../04-validation/go-no-go-criteria.md). Do not begin Phase 2 without passing Phase 1's gate.

## Related

- [RFC](../03-rfc/rfc-001-agent-harness.md) — technical architecture that implements these requirements
- [Specs](../05-specs/index.md) — formal specifications for the language and runtime
- [Design Docs](../06-design/index.md) — system design for each component
