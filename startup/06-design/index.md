---
type: Index
title: "Design Docs Index"
description: Navigation index for the AgentForge system design documents. Covers system architecture, compiler design, runtime design, GPU serving, billing, and data model.
tags: [design, index, architecture, okf]
timestamp: 2026-06-28T00:00:00Z
---

# Design Docs

Component-level system designs with diagrams, schemas, and implementation notes. These documents bridge the formal specs and the implementation tasks in the PRD.

## Documents

| Document | Description |
|---|---|
| [system-architecture.md](system-architecture.md) | Full component diagram, data flow, external integrations |
| [compiler-design.md](compiler-design.md) | Lexer → Parser → AST → Symbol Table → Type Checker → IR emitter |
| [runtime-design.md](runtime-design.md) | IR walker, model router, tool executor, context budget manager |
| [gpu-serving-design.md](gpu-serving-design.md) | vLLM + Gemma 4 26B-A4B MTP speculative decoding stack |
| [billing-design.md](billing-design.md) | Stripe metered billing, Bedrock pass-through, credit system |
| [data-model.md](data-model.md) | Postgres schema, tenant isolation, migration strategy |

## Reading Order

For a new engineer or coding agent starting implementation:

1. Read [system-architecture.md](system-architecture.md) for the big picture
2. Read [compiler-design.md](compiler-design.md) before implementing Phase 1A tasks
3. Read [runtime-design.md](runtime-design.md) before implementing Phase 1B tasks
4. Read [data-model.md](data-model.md) before implementing Phase 1D tasks
5. Read [billing-design.md](billing-design.md) before implementing TASK-1-18
6. Read [gpu-serving-design.md](gpu-serving-design.md) before implementing Phase 3A tasks

## Related

- [Specs](../05-specs/index.md) — formal specs that these designs implement
- [RFC-001](../03-rfc/rfc-001-agent-harness.md) — the architecture proposal these designs elaborate
- [PRD](../02-prd/index.md) — task decomposition that implements these designs
