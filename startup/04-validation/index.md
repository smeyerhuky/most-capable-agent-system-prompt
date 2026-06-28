---
type: Index
title: "Validation Index"
description: Navigation index for the AgentForge validation section. Contains phase go/no-go criteria and pre-build validation experiments.
tags: [validation, index, go-no-go, experiments, okf]
timestamp: 2026-06-28T00:00:00Z
---

# Validation

Before writing code, validate assumptions. Before beginning each phase, pass the gate. This section defines both.

## Documents

| Document | Description |
|---|---|
| [go-no-go-criteria.md](go-no-go-criteria.md) | Numeric phase gates with explicit thresholds — passed before each phase begins |
| [experiments.md](experiments.md) | The 5 pre-build experiments to validate before writing any production code |

## Philosophy

> "If you can't measure it, you can't manage it. If you haven't defined the measurement before you build, you'll rationalize success afterward."

Every phase gate is defined in advance with observable, numeric criteria. When the gate is reached, a pass/fail decision is made in writing and recorded in [log.md](../log.md).

## Related

- [PR/FAQ Internal FAQ](../01-prfaq/internal-faq.md) — assumptions that these experiments test
- [PRD Overview](../02-prd/overview.md) — success metrics these gates enforce
- [Business Plan](../07-business-plan/index.md) — market assumptions being tested
