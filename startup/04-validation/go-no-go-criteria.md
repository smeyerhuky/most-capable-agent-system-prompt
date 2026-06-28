---
type: Validation
title: "AgentForge Go/No-Go Criteria"
description: Numeric phase gates for all three AgentForge delivery phases. Each gate must be explicitly passed in writing before the next phase begins. Defines the observable thresholds for product-market fit, technical readiness, and financial health.
tags: [validation, go-no-go, phase-gates, metrics, criteria]
timestamp: 2026-06-28T00:00:00Z
---

# AgentForge Go/No-Go Criteria

Each phase gate is a binary decision: **GO** or **NO-GO**. All criteria in a gate must pass for a GO decision. A NO-GO means: investigate, iterate, or pivot before proceeding. Record the decision and rationale in [log.md](../log.md).

---

## Phase 1 Gate

**Decision required by:** End of Month 2

**Proceed to Phase 2 only if ALL of the following pass:**

### Product Criteria

| # | Criterion | Threshold | Measurement |
|---|---|---|---|
| P1.1 | Paying users | ≥ 10 at $29/month | Stripe dashboard |
| P1.2 | Time-to-first-run (new user, unassisted) | ≤ 15 minutes | Measured on 5 real users via screen share |
| P1.3 | Compiler false-positive rate | < 5% | Compiler telemetry: valid programs rejected / total programs compiled |
| P1.4 | Runtime uptime | ≥ 99.0% over 14-day window | AWS CloudWatch availability metric |
| P1.5 | User retention (day 7) | ≥ 40% | Users who ran a second pipeline within 7 days of first run |

### Technical Criteria

| # | Criterion | Threshold | Measurement |
|---|---|---|---|
| T1.1 | Bedrock billing accuracy | 100% (zero discrepancies) | Monthly reconciliation: Stripe charges vs. AWS Cost Explorer |
| T1.2 | CI test pass rate | ≥ 95% | GitHub Actions CI on `main` |
| T1.3 | API P95 latency (non-LLM endpoints) | < 500ms | CloudWatch |
| T1.4 | Run recovery rate | ≥ 90% | Failed runs that successfully resume from last checkpoint / total failures |

### Business Criteria

| # | Criterion | Threshold | Measurement |
|---|---|---|---|
| B1.1 | MRR | ≥ $290 (10 × $29) | Stripe |
| B1.2 | Churn rate (month 1→2) | < 30% | (Churned subscribers) / (Total subscribers at start of month) |
| B1.3 | Support tickets resolved < 24h | ≥ 90% | Helpdesk |

### NO-GO Remediation Paths

| Failed Criterion | Likely Root Cause | Remediation |
|---|---|---|
| P1.2 (time-to-first-run > 15 min) | Onboarding friction | Simplify onboarding to 1-step (auto-generate first pipeline from NL description) |
| P1.1 (< 10 paying users) | No distribution yet | Manually install for 10 developers (Collison installation method) |
| T1.1 (billing discrepancy) | Bedrock token counting mismatch | Do not proceed until billing is accurate — trust is irrecoverable |
| P1.5 (day-7 retention < 40%) | Value not realized on first run | Interview 5 users who did not return; iterate on onboarding |

---

## Phase 2 Gate

**Decision required by:** End of Month 4

**Proceed to Phase 3 only if ALL of the following pass:**

### Product Criteria

| # | Criterion | Threshold | Measurement |
|---|---|---|---|
| P2.1 | GitHub stars (open-source compiler) | ≥ 500 | GitHub |
| P2.2 | VS Code extension installs | ≥ 1,000 | VS Code Marketplace |
| P2.3 | Monthly active pipelines | ≥ 100 | Pipelines with ≥ 1 run in last 30 days |
| P2.4 | MCP tool integrations tested | ≥ 3 third-party MCP servers | Integration test suite |
| P2.5 | AI-generator success rate | ≥ 80% compile pass on generated pipelines | Compiler telemetry on AI-generated files |

### Business Criteria

| # | Criterion | Threshold | Measurement |
|---|---|---|---|
| B2.1 | MRR | ≥ $2,900 (100 × $29) | Stripe |
| B2.2 | Month-over-month MRR growth | ≥ 20% | Stripe |
| B2.3 | Organic traffic to landing page | ≥ 500 unique visitors/month | Analytics |
| B2.4 | Template marketplace submissions | ≥ 3 community-submitted templates | Marketplace database |

### NO-GO Remediation Paths

| Failed Criterion | Likely Root Cause | Remediation |
|---|---|---|
| P2.1 (< 500 stars) | HN post did not land | Try Product Hunt launch, Twitter thread, re-post HN with better title |
| B2.1 (MRR < $2,900) | Free tier too generous / conversion too low | Tighten free tier to 100 executions; add "upgrade to unlock" prompt at limit |
| P2.3 (< 100 active pipelines) | Retention problem | Scheduled pipeline feature (pipelines that run on cron) to increase stickiness |

---

## Phase 3 Gate {#phase-3-gate}

**Decision required by:** End of Month 6

**Proceed to fundraising / hiring only if ALL of the following pass:**

### Product Criteria

| # | Criterion | Threshold | Measurement |
|---|---|---|---|
| P3.1 | Private GPU tier adoption | ≥ 20% of paid-tier LLM calls on GPU tier | Billing system |
| P3.2 | Team workspace accounts | ≥ 5 teams | Auth system |
| P3.3 | Enterprise pilot customers | ≥ 2 (on SSO + audit logs) | CRM |
| P3.4 | AWS Marketplace listing live | Yes | AWS Partner Network |
| P3.5 | GPU tier uptime | ≥ 98% (lower than Bedrock due to spot) | CloudWatch |

### Business Criteria

| # | Criterion | Threshold | Measurement |
|---|---|---|---|
| B3.1 | MRR | ≥ $10,000 | Stripe |
| B3.2 | Net Revenue Retention (NRR) | ≥ 110% | (MRR at end of month from cohort) / (MRR at start of month from same cohort) |
| B3.3 | CAC payback period | ≤ 6 months | (Sales + Marketing spend) / (New MRR added) |
| B3.4 | Gross margin | ≥ 60% | (Revenue − COGS) / Revenue |

### NO-GO Remediation Paths

| Failed Criterion | Likely Root Cause | Remediation |
|---|---|---|
| B3.1 (MRR < $10,000) | Missing enterprise motion | Hire first sales person (SDR) earlier than planned |
| P3.3 (< 2 enterprise pilots) | No enterprise outreach | Cold outreach to 50 engineering managers at 50–500 person companies |
| B3.3 (CAC payback > 6 months) | Over-spending on paid acquisition | Refocus on PLG: improve template marketplace, in-product sharing, GitHub Actions |

---

## Related

- [Validation Experiments](experiments.md) — pre-build experiments that feed into these gates
- [PRD Overview: Success Metrics](../02-prd/overview.md) — the metrics these gates enforce
- [Revenue Model](../07-business-plan/revenue-model.md) — pricing definitions underlying the financial criteria
