---
type: Startup Bundle
title: "AgentForge — Startup in a Box"
description: Root index for the AgentForge founder documentation suite. AgentForge is a DSL compiler and cloud runtime for LLM-backed agentic workflows. This bundle contains the complete set of planning, technical, and business documents required to build and launch the company.
tags: [agentforge, startup, index, okf, agentic-harness, dsl]
timestamp: 2026-06-28T00:00:00Z
---

# AgentForge — Startup in a Box

AgentForge is a programming language and cloud runtime for AI agent workflows. Developers write `.agent` files — a high-level DSL that compiles to a structured IR — and run them on a managed cloud that routes to Anthropic (via AWS Bedrock), OpenAI, or a private self-hosted Gemma 4 GPU tier. The harness handles context management, tool dispatch, model routing, billing, and observability so the developer only writes intent.

## Type Vocabulary

All files in this bundle declare one of the following producer-defined types:

| Type | Meaning |
|---|---|
| Startup Bundle | The bundle root. Declares type vocabulary and links all sections. |
| Index | A directory `index.md` navigation entry point. |
| Log | Append-only record of document generations and edits. |
| PRFAQ Section | A sub-document of the Press Release + FAQ. |
| PRD Overview | Phase map, design principles, and success metrics. |
| PRD Phase | A single delivery phase with agentic PDLC task decomposition. |
| RFC | Request for Comments — technical architecture proposal open for review. |
| Validation | Go/no-go criteria or validation experiment descriptions. |
| Spec | Formal specification — grammar, API surface, or runtime contract. |
| Design Doc | Component-level system design with diagrams and schemas. |
| Business Plan Section | A named section of the business plan. |

## Bundle Structure

```
startup/
├── index.md                        ← you are here
├── log.md                          ← edit history
│
├── 01-prfaq/                       ← customer-first product definition
├── 02-prd/                         ← phased requirements + agentic PDLC tasks
├── 03-rfc/                         ← technical architecture RFC
├── 04-validation/                  ← go/no-go gates + pre-build experiments
├── 05-specs/                       ← formal DSL, runtime, and API specs
├── 06-design/                      ← system design docs
└── 07-business-plan/               ← full startup business plan
```

## Navigation

| Section | Description | Entry Point |
|---|---|---|
| PR/FAQ | Amazon-style press release + internal hard questions | [01-prfaq/index.md](01-prfaq/index.md) |
| PRD | Phased product requirements, agentic PDLC task decomposition | [02-prd/index.md](02-prd/index.md) |
| RFC | Technical architecture proposal for the compiler + runtime | [03-rfc/index.md](03-rfc/index.md) |
| Validation | Phase gates and pre-build experiments | [04-validation/index.md](04-validation/index.md) |
| Specs | Formal language, runtime, and API specifications | [05-specs/index.md](05-specs/index.md) |
| Design Docs | Component diagrams, schemas, GPU serving, billing | [06-design/index.md](06-design/index.md) |
| Business Plan | Market, GTM, sales, revenue, financial model | [07-business-plan/index.md](07-business-plan/index.md) |

## Founder Context

**Founder:** Sole technical founder, 30 years of experience across architecture, design, testing, infrastructure, and developer experience.

**Working name:** AgentForge

**DSL file extension:** `.agent`

**Core insight:** The `.agent` language is the moat. Competitors compete on model quality; AgentForge competes on portability, auditability, and compile-time correctness of agent workflows.

**Primary comparable:** n8n (open-core, PLG, execution-based billing). Secondary: Lovable (credits + subscription). Technical foundation: classical compiler pipeline (lexer → parser → AST → IR → runtime) applied to agent orchestration.
