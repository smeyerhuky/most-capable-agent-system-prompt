---
type: Validation
title: "AgentForge Go/No-Go Criteria"
description: Numeric phase gates for AgentForge, re-baselined onto a realistic solo-founder timeline. Adds a dedicated DSL-stickiness gate (the #1 risk for a new language), demotes GitHub stars from a binary gate to a health signal, and replaces single-event vanity thresholds with adoption-depth metrics.
tags: [validation, go-no-go, phase-gates, metrics, dsl-stickiness, timeline]
timestamp: 2026-06-28T00:00:00Z
---

# AgentForge Go/No-Go Criteria

Each gate is a **GO** or **NO-GO** decision. All criteria must pass for GO. A NO-GO means investigate, iterate, or pivot before proceeding. Record decisions in [log.md](../log.md).

---

## Timeline re-baseline (read first)

The prior gates assumed Phase 1 closed at **Month 2** with a working compiler, multi-tenant cloud, billing, a VS Code extension, and 10 paying users — built solo. That is a 12–18 month scope compressed into 8 weeks. Gates are re-baselined to a realistic solo build:

| Gate | Old date | New date | Why |
|---|---|---|---|
| Phase 1 | Month 2 | **Month 6** | Compiler + runtime + multi-tenant cloud + billing + VS Code ext is a 5–6 month build before the first paying user |
| Phase 2 | Month 4 | **Month 12** | Community + AI-generator + MCP integrations need real adoption time |
| Phase 3 | Month 6 | **Month 18** | Private GPU tier + enterprise pilots are an 18-month-in milestone |

The PRD phase week-numbers (`02-prd/`) assume uninterrupted full-time focus and should be re-baselined to match these dates.

---

## Phase 1 Gate — Decision by end of Month 6

**Proceed to Phase 2 only if ALL pass:**

### Product

| # | Criterion | Threshold | Measurement |
|---|---|---|---|
| P1.1 | Paying users | ≥ 10 at $29/mo | Stripe |
| P1.2 | Time-to-first-run (new user, unassisted) | ≤ 15 min | 5 real users, screen share |
| P1.3 | Compiler false-positive rate | < 5% | valid programs rejected / total compiled |
| P1.4 | Runtime uptime | ≥ 99.0% over 14 days | CloudWatch |
| P1.5 | Day-7 retention | ≥ 40% | ran a 2nd pipeline within 7 days |

### Technical

| # | Criterion | Threshold | Measurement |
|---|---|---|---|
| T1.1 | Bedrock billing accuracy | 100% (zero discrepancies) | Stripe vs AWS Cost Explorer |
| T1.2 | CI test pass rate | ≥ 95% | GitHub Actions on `main` |
| T1.3 | API P95 latency (non-LLM) | < 500ms | CloudWatch |
| T1.4 | Run recovery rate | ≥ 90% | failed runs resumed from checkpoint / total failures |

### Business

| # | Criterion | Threshold | Measurement |
|---|---|---|---|
| B1.1 | MRR | ≥ $290 | Stripe |
| B1.2 | Month-1→2 churn | < 30% | churned / start-of-month subscribers |
| B1.3 | Support tickets resolved < 24h | ≥ 90% | Helpdesk |

---

## Phase 1.5 Gate — DSL Stickiness — Decision by end of Month 9

**New gate.** A new programming language dies when the novelty wears off and users drift back to Python/TS. Day-7 retention catches early drop-off; it does **not** catch whether people are *investing* in the language. This gate does. It is a hard gate: **a NO-GO here means the DSL is a demo, not a platform — fix it before spending on growth.**

| # | Criterion | Threshold | Why it matters |
|---|---|---|---|
| S1.1 | Week-4 author retention | ≥ 25% of Week-1 authors still writing `.agent` files in Week 4 | Sustained authorship, not one-time curiosity |
| S1.2 | Returning-author rate | ≥ 35% of paying users author a pipeline in ≥ 3 distinct weeks/month | Habit formation |
| S1.3 | Pipeline-depth growth | median step-count of a user's pipelines rises ≥ 20% from first to fourth week | Users are expressing *more* in the DSL over time, not less |
| S1.4 | Beyond-the-template rate | ≥ 50% of active pipelines diverge materially from a starter template | They're writing their own programs, not just running samples |
| S1.5 | Model-portability usage | ≥ 30% of multi-step pipelines reference ≥ 2 model aliases | The portability moat is actually being used |

### Stickiness NO-GO remediation

| Failed | Likely cause | Remediation |
|---|---|---|
| S1.1 / S1.2 | DSL friction exceeds value once novelty fades | Ship LSP quick-fixes, better error messages, and a `.agent` formatter; interview 5 lapsed authors |
| S1.3 | Users hit an expressiveness ceiling | Prioritize `tool_def` blocks and `loop parallel:` (RFC open questions #1, #2) |
| S1.5 | Portability is theoretical | Publish side-by-side cost/quality diffs per model; make model-swap a one-click dashboard action |

---

## Phase 2 Gate — Decision by end of Month 12

**Proceed to Phase 3 only if ALL pass.** Note: GitHub stars are now a **health signal, not a gate** — a single HN/PH outcome must not be a company-continuation decision. The real gates are adoption-depth and revenue.

### Product

| # | Criterion | Threshold | Measurement |
|---|---|---|---|
| P2.1 | Monthly active pipelines | ≥ 100 | ≥ 1 run in last 30 days |
| P2.2 | VS Code extension installs | ≥ 1,000 | Marketplace |
| P2.3 | MCP tool integrations tested | ≥ 3 third-party servers | Integration suite |
| P2.4 | AI-generator success rate | ≥ 80% compile pass | Compiler telemetry on generated files |
| P2.5 | Net Revenue Retention | ≥ 100% | cohort MRR end / start |

### Business

| # | Criterion | Threshold | Measurement |
|---|---|---|---|
| B2.1 | MRR | ≥ $3,000 | Stripe |
| B2.2 | Net MoM MRR growth (3-mo avg, **net of churn**) | ≥ 12% | Stripe |
| B2.3 | Organic landing-page traffic | ≥ 500 uniques/mo | Analytics |
| B2.4 | Community template submissions | ≥ 3 | Marketplace DB |

### Health signals (tracked, not gated)

| Signal | Watch level |
|---|---|
| GitHub stars | 500+ is healthy; below is a distribution problem to fix, not a reason to stop |
| HN/Product Hunt rank | informs marketing, not the gate |

### NO-GO remediation

| Failed | Likely cause | Remediation |
|---|---|---|
| B2.2 (net growth < 12%) | Churn eating gross adds | Attack Builder churn first (it's 43% of mix and the leakiest); revisit Phase-1.5 stickiness levers |
| P2.1 (< 100 active pipelines) | Retention/stickiness | Ship scheduled (cron) pipelines to raise recurring runs |
| B2.1 (MRR < $3,000) | Free tier too generous / weak conversion | Tighten free tier; add upgrade prompt at the execution limit |

---

## Phase 3 Gate — Decision by end of Month 18

**Proceed to fundraising / scaled hiring only if ALL pass:**

### Product

| # | Criterion | Threshold | Measurement |
|---|---|---|---|
| P3.1 | Private-tier adoption (privacy-driven) | ≥ 20% of paid LLM calls on the private/VPC tier | Billing |
| P3.2 | Team workspace accounts | ≥ 5 teams | Auth |
| P3.3 | Enterprise pilots (SSO + audit logs + VPC) | ≥ 2 | CRM |
| P3.4 | AWS Marketplace listing live | Yes | APN |
| P3.5 | GPU/private-tier uptime | ≥ 98% | CloudWatch |
| P3.6 | GPU build-vs-buy break-even cleared | Yes/No | sustained utilization > break-even in gpu-serving-design.md; if No, keep reselling hosted Gemma |

### Business

| # | Criterion | Threshold | Measurement |
|---|---|---|---|
| B3.1 | MRR | ≥ $10,000 | Stripe |
| B3.2 | Net Revenue Retention | ≥ 110% | cohort MRR end / start |
| B3.3 | CAC payback | ≤ 6 months | S&M spend / new MRR |
| B3.4 | Gross margin (subscription) | ≥ 80% | (Rev − COGS) / Rev, pass-through excluded |
| B3.5 | Cumulative burn vs plan | within 15% of the ~$178K trough | Bookkeeping |

### NO-GO remediation

| Failed | Likely cause | Remediation |
|---|---|---|
| P3.6 (GPU break-even not cleared) | Fleet under-utilized | Do not build the fleet; resell hosted Gemma — it's cheaper until utilization justifies owning |
| B3.1 (MRR < $10K) | Missing enterprise motion | Cold outreach to 50 eng managers at 50–500-person cos; lead with VPC/privacy, not price |
| P3.3 (< 2 pilots) | No enterprise outreach | Same; the enterprise wedge is data-never-leaves-VPC |

---

## Related

- [Validation Experiments](experiments.md)
- [Financial Model](../07-business-plan/financial-model.md) — churn-netted ramp these gates enforce
- [GPU Serving Design](../06-design/gpu-serving-design.md) — the build-vs-buy break-even behind P3.6
- [Revenue Model](../07-business-plan/revenue-model.md)
