---
type: Log
title: "AgentForge Documentation Log"
description: Append-only record of all document generations and significant edits to the AgentForge startup bundle.
tags: [log, history, okf]
timestamp: 2026-06-28T00:00:00Z
---

# AgentForge Documentation Log

Append new entries at the bottom. Do not edit previous entries.

---

## 2026-06-28T00:00:00Z — Initial Bundle Generation

Generated the full AgentForge "Startup in a Box" documentation suite (37 files) from a founder design session. Session covered:

- DSL design and compilation model (`.agent` files, lexer → parser → AST → IR → runtime dispatch)
- Gemma 4 MTP speculative decoding architecture and GPU cost analysis
- Business model analysis (n8n, Lovable, Cursor, Windsurf/Codeium comparables)
- YC best practices research for sole technical founders
- AWS Bedrock + ISV Accelerate monetization path
- Three-tier model strategy (Bedrock Claude / Gemma 4-26B-A4B private / Gemma 4-12B fast)
- OKF-compliant structure with full type vocabulary

Files created: 37 (index + log + 7 section indexes + 28 content documents)

Author: Founder session via Claude Code

---

## 2026-06-28T00:00:00Z — Financial Realism + Validation + GPU Repositioning Revision

Revised three areas after a critical review of the bundle's internal consistency:

1. **Financial model rebuilt** (`07-business-plan/financial-model.md`): revenue now starts Month 6 (realistic solo build), churn is netted into every month, and the three conflicting Year-3 ARR figures ($647K / $2.7M / $3.6M) are reconciled into one base case (~$664K) plus explicitly-labeled seed-accelerated (~$1.2M) and aggressive (~$3.6M) cases. Corrected break-even: ~Month 25; peak cash trough ~$178K (was claimed Month 9 / $66K).

2. **Validation gates re-baselined** (`04-validation/go-no-go-criteria.md`): gate dates moved to Month 6/12/18; added a dedicated DSL-stickiness gate (Phase 1.5: week-4 author retention, returning-author rate, pipeline-depth growth, beyond-template rate, model-portability usage); demoted GitHub stars from a binary gate to a health signal; added GPU build-vs-buy break-even gate (P3.6).

3. **GPU tier repositioned** (`06-design/gpu-serving-design.md`, `07-business-plan/revenue-model.md`, `executive-summary.md`, `market-analysis.md`): the private tier's moat is now framed as privacy/VPC control, not price. Added build-vs-buy unit economics (owned fleet only beats hosted Gemma's $0.33/1M at high batching utilization). Corrected the inconsistent "10× / 100× cheaper" claims to the defensible ~45×-vs-frontier-Bedrock figure.

Author: Critical review revision via Claude

---

## 2026-06-28T22:13:32Z — executive-summary Use-of-Funds Reconciliation

**Why:** PR review comment r3488613241 identified that the "Use of funds" bullet list in
`executive-summary.md` was inconsistent with the rebuilt `financial-model.md` table. Specifically:
- The bullet list stated 60% founder salary, implying ~$300K at $10K/mo × 18 months, whereas the
  financial model correctly shows $180K = 36%.
- DevRel/Community hire ($120K, 24%) and Infra engineer ($60K, 12%) were missing entirely.
- Infrastructure, marketing, and legal percentages were stale rounded approximations.

**How:** Replaced the 4-bullet list with a Markdown table that mirrors `financial-model.md:88-95`
line-for-line, including the same category names, dollar amounts, and percentages. Added an HTML
comment above the table to explain the provenance of the change so future editors can trace it.

Files changed: `startup/07-business-plan/executive-summary.md`

Author: Copilot coding agent (r3488613241 fix)
