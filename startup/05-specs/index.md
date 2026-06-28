---
type: Index
title: "Specs Index"
description: Navigation index for the AgentForge formal specifications. Contains the language spec, runtime spec, and API spec.
tags: [specs, index, language, runtime, api, okf]
timestamp: 2026-06-28T00:00:00Z
---

# Specifications

Formal specifications are the authoritative source of truth for the AgentForge language, runtime, and API. Implementation must conform to these specs. Tests must cover all spec behaviors. The specs are public and versioned.

## Documents

| Document | Description |
|---|---|
| [language-spec.md](language-spec.md) | `.agent` DSL: EBNF grammar, type system, built-in stdlib, error codes |
| [runtime-spec.md](runtime-spec.md) | IR instruction set, execution model, tool contract, event stream |
| [api-spec.md](api-spec.md) | REST API endpoints, WebSocket events, OpenAI-compatible model endpoint |

## Versioning

All specs are versioned independently. The current version of each spec is declared in its frontmatter. Breaking changes require a new major version. Additive changes are minor versions.

| Spec | Current Version |
|---|---|
| Language Spec | 0.1.0 |
| Runtime Spec | 0.1.0 |
| API Spec | 0.1.0 |

## Related

- [RFC-001](../03-rfc/rfc-001-agent-harness.md) — the architecture proposal these specs implement
- [Design Docs](../06-design/index.md) — component designs that implement these specs
- [PRD Phase 1](../02-prd/phase-1-mvp.md) — tasks that implement these specs
