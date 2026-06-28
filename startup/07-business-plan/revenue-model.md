---
type: Business Plan Section
title: "AgentForge Revenue Model"
description: Pricing tier definitions, unit economics, credit system, pass-through billing mechanics, and LTV/CAC analysis for AgentForge.
tags: [business-plan, revenue-model, pricing, unit-economics, ltv, cac, credits, billing]
timestamp: 2026-06-28T00:00:00Z
---

# AgentForge Revenue Model

---

## Pricing Tiers

### Hobbyist — Free

**Who it's for:** Individual developers exploring AgentForge, students, open-source projects.

| Feature | Included |
|---|---|
| Runs per month | 500 |
| Pipeline storage | 10 pipelines |
| Models | Community models only (Gemma 4-12B fast tier) |
| Context budget per run | 8,000 tokens |
| Dashboard | Basic (no cost breakdown) |
| API access | No |
| Support | Community Discord only |

**Why this tier exists:** Free users become paid users. Free users share pipelines that bring new users. Free users file bug reports. The free tier is not a charity — it's acquisition.

**Free tier economics:** Average Hobbyist costs ~$0.002/month in compute (Gemma 4-12B on shared GPU). At 1,000 free users: $2/month cost. Negligible.

---

### Builder — $29/month

**Who it's for:** Individual professional developers using AgentForge for production work.

| Feature | Included |
|---|---|
| Runs per month | 5,000 |
| Pipeline storage | Unlimited |
| Models | All (Bedrock Claude, Llama 3, GPT-4o, Gemma 4-26B-A4B private) |
| GPU tier | 1M tokens/month included |
| Bedrock billing | Pass-through at cost |
| Context budget per run | 32,000 tokens |
| Dashboard | Full (cost breakdown, model comparison, MTP stats) |
| API access | Yes |
| VS Code extension | Full Intellisense + one-click run |
| Support | Email + Discord |

**COGS per Builder user per month:**
- Infrastructure (hosting, DB, Redis): ~$0.50
- GPU tier (1M tokens at $0.10/1M amortized): $0.10
- Support (amortized): ~$1.50
- AWS fees: ~$0.20
- **Total COGS: ~$2.30**
- **Gross margin: ($29 - $2.30) / $29 = 92%** (excluding Bedrock pass-through, which is zero-margin by design)

---

### Team — $99/seat/month

**Who it's for:** Engineering teams (3–50 developers) building agent workflows together.

| Feature | Included (per seat) |
|---|---|
| Runs per month | Unlimited |
| GPU tier | 5M tokens/seat/month included |
| Bedrock billing | Pass-through at cost |
| Team workspaces | Shared pipelines, role-based access |
| Pipeline templates | Shared team library |
| CI/CD integration | GitHub Actions + GitLab CI |
| Audit history | 30-day run history |
| Context budget per run | 128,000 tokens |
| Support | Priority email + dedicated Slack channel (5+ seats) |

**Minimum team size:** 1 seat (solo developer can use Team plan for unlimited runs)

**COGS per Team seat per month:**
- Infrastructure: ~$1.50
- GPU tier (5M tokens): ~$0.50
- Support: ~$3.00
- **Total COGS: ~$5.00**
- **Gross margin: ($99 - $5) / $99 = 95%** (excluding pass-through)

---

### Enterprise — from $2,000/month

**Who it's for:** Companies with compliance requirements, dedicated GPU allocations, and legal/procurement processes.

| Feature | Included |
|---|---|
| Seats | Negotiated (usually 10–50) |
| GPU tier | Dedicated allocation |
| Bedrock billing | Pass-through at cost OR private Bedrock account |
| SSO | SAML 2.0 (Okta, Azure AD, Google Workspace) |
| VPC deployment | Run AgentForge control plane in customer VPC |
| Audit logs | 90-day retention, exportable |
| SLA | 99.9% uptime, 4-hour response on P1 |
| Custom models | Bring your own fine-tuned models |
| Onboarding | Dedicated engineer, 30-day setup support |

**COGS per Enterprise customer at $2,000/month:**
- Infrastructure (dedicated): ~$200
- GPU dedicated allocation: ~$250–500
- Support/CS (10% of time): ~$200
- **Total COGS: ~$650–900**
- **Gross margin: 55–67%** (lower than other tiers due to higher support cost — still healthy)

---

## Bedrock Pass-Through Mechanics

Every dollar of Bedrock token spend is passed through to users at exact cost. This is a deliberate design choice that:

1. Builds trust (users can verify by checking their own AWS Cost Explorer)
2. Eliminates billing risk (we never lose money on model inference)
3. Is a competitive differentiator ("zero markup" is a marketing asset)
4. Enables a large-volume relationship with AWS (we process significant Bedrock spend without it being our cost — good for ISV Accelerate)

**The revenue from pass-through:** $0 direct. But users who trust our billing stay on the subscription. The subscription is the revenue.

---

## Credit System

**Bedrock credits:** Not applicable — Bedrock is pure pass-through billed on the monthly invoice.

**GPU tier credits:** Included monthly allocation of GPU tokens:
- Builder: 1M tokens/month
- Team: 5M tokens/seat/month
- Enterprise: Negotiated

**Overage pricing:**
- Builder overage: $0.10/1M GPU tokens (2× cost, small margin)
- Team overage: $0.08/1M GPU tokens (1.6× cost, volume discount)

**Credit rollover:** No. Credits reset on the 1st of each month. Unused credits do not carry over (standard SaaS practice — prevents balance sheet liability).

---

## Unit Economics

### LTV (Lifetime Value)

| Tier | Monthly Revenue | Avg Lifetime | LTV |
|---|---:|---:|---:|
| Builder | $29 | 18 months | $522 |
| Team (5 seats) | $495 | 30 months | $14,850 |
| Enterprise | $2,500 avg | 48 months | $120,000 |

Assumptions:
- Builder churn: 5.5%/month (industry average for SMB SaaS)
- Team churn: 3%/month (stickier due to team coordination cost)
- Enterprise churn: 2%/month (lowest due to SSO/VPC lock-in)

### CAC (Customer Acquisition Cost)

Phase 1–2 target (PLG, minimal spend):
- Builder: $0 CAC (organic, self-serve)
- Team: $200 CAC (1–2 hours founder time for installation + demo)
- Enterprise: $3,000 CAC (multiple demo calls, pilot setup, legal)

Phase 3+ with first sales hire:
- Builder: $50 CAC (content + VS Code discovery)
- Team: $500 CAC (sales hire amortized)
- Enterprise: $8,000 CAC (full sales cycle)

### CAC Payback Period

| Tier | LTV | CAC (Phase 3) | Payback |
|---|---:|---:|---:|
| Builder | $522 | $50 | 2 months |
| Team | $14,850 | $500 | 1 month |
| Enterprise | $120,000 | $8,000 | 3.8 months |

All tiers have payback periods well under 12 months — strong SaaS unit economics.

### LTV:CAC Ratio

| Tier | LTV:CAC |
|---|---:|
| Builder | 10.4× |
| Team | 29.7× |
| Enterprise | 15× |

Target for SaaS: > 3×. AgentForge is well above target on all tiers due to PLG-driven low CAC.

---

## Revenue Mix Target (Year 3)

| Tier | Users | MRR | % of MRR |
|---|---:|---:|---:|
| Builder | 800 | $23,200 | 43% |
| Team | 50 teams (avg 5 seats) | $24,750 | 46% |
| Enterprise | 3 | $6,000 | 11% |
| **Total** | | **$53,950** | **100%** |

Annual recurring revenue (Year 3): **~$647K ARR**. This is a conservative, default-alive projection. With a seed round and first sales hire: potential $2–3M ARR by Year 3.

---

## Related

- [Financial Model](financial-model.md) — 3-year P&L built on these unit economics
- [Billing Design](../06-design/billing-design.md) — how the billing system implements this model
- [GPU Serving Design](../06-design/gpu-serving-design.md) — cost basis for GPU tier pricing
