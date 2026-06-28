---
type: Business Plan Section
title: "AgentForge Financial Model"
description: 3-year P&L, monthly cash burn, break-even analysis, and funding scenarios for AgentForge. Rebuilt on a realistic solo-founder build timeline with churn netted out of the revenue ramp. Base case is the planning number; the seed-accelerated and aggressive cases are explicitly labeled as upside.
tags: [business-plan, financial-model, pl, cash-burn, break-even, funding, runway, churn, base-case]
timestamp: 2026-06-28T00:00:00Z
---

# AgentForge Financial Model

All figures in USD. US-based sole founder. This model was rebuilt to fix three errors in the prior version: (1) it assumed paying revenue in Month 1–2, which is impossible for a from-scratch compiler + multi-tenant runtime + billing build; (2) it treated monthly MRR adds as net, ignoring churn; (3) it carried three different Year-3 ARR figures across the bundle. This version starts revenue at Month 6, nets churn out of every month, and reconciles to a single base case.

---

## Reconciliation note (read first)

The bundle previously stated three different Year-3 ARR numbers: ~$648K (revenue-model.md, bottoms-up), $2.7M (market-analysis.md, SOM), and $3.6M (this file, top-line). They are now reconciled into one base case and two clearly-labeled upside cases:

| Case | Year-3 ARR | What it assumes |
|---|---:|---|
| **Base (planning number)** | **~$664K** | Realistic solo build, revenue from Month 6, churn netted, PLG-only acquisition |
| Seed-accelerated (upside) | ~$1.2M | $500K seed funds earlier DevRel + infra hire; faster, sustained adds |
| Aggressive (ceiling) | ~$3.6M | Cursor-class viral PLG; possible, not plannable. Do not budget against this. |

The base case is independently derived (churn-netted model below) and lands within ~2% of the bottoms-up figure in [revenue-model.md](revenue-model.md) — that convergence is the reason it is the planning number.

---

## Key Assumptions

| Assumption | Value | Source / rationale |
|---|---|---|
| Build phase (no revenue) | Months 1–5 | Compiler pipeline + runtime + multi-tenant cloud + Stripe/Bedrock billing + VS Code ext is a 5–6 month solo build before first paying user |
| First paying cohort | Month 6, ~12 users | Design-partner conversion off manual ("do things that don't scale") outreach |
| Blended monthly logo churn | 5.0% (Y1) → 4.2% (Y2) → 3.6% (Y3) | Builder-heavy early; declines as stickier Team/Enterprise mix grows. Netted out of every month. |
| Blended ARPU | $31 (Y1) → $39 (Y2) → $47 (Y3) | Derived from tier mix, not assumed flat. Builder $29 dominates early; Team seats lift the blend. |
| Gross margin (subscription) | 90% | Bedrock pass-through is zero-margin by design and excluded |
| Founder salary | $120K/yr ($10K/mo) | Conservative |
| Infrastructure | ~$800/mo base + ~4% of MRR | AWS ECS + RDS + Redis + S3, scales mildly with usage |
| First hire | Month 18 (DevRel, $7K/mo) | Only once MRR can partly support it — not Month 7 |

---

## Monthly P&L — Base Case (Bootstrap, no external funding)

90% gross margin on subscription. Churn netted into the customer count each month.

| Month | Paying customers | MRR | Opex | Net | Cumulative |
|---:|---:|---:|---:|---:|---:|
| 1 | 0 | $0 | $10,800 | -$10,800 | -$10,800 |
| 2 | 0 | $0 | $10,800 | -$10,800 | -$21,600 |
| 3 | 0 | $0 | $10,800 | -$10,800 | -$32,400 |
| 4 | 0 | $0 | $10,800 | -$10,800 | -$43,200 |
| 5 | 0 | $0 | $10,800 | -$10,800 | -$54,000 |
| 6 | 12 | $372 | $10,815 | -$10,480 | -$64,480 |
| 7 | 25 | $774 | $10,831 | -$10,134 | -$74,614 |
| 8 | 39 | $1,210 | $10,848 | -$9,760 | -$84,374 |
| 9 | 54 | $1,682 | $10,867 | -$9,354 | -$93,728 |
| 10 | 71 | $2,193 | $10,888 | -$8,914 | -$102,642 |
| 11 | 89 | $2,748 | $10,910 | -$8,437 | -$111,078 |
| 12 | 108 | $3,349 | $10,934 | -$7,920 | -$118,998 |
| 15 | 179 | $6,980 | $11,079 | -$4,797 | -$135,893 |
| 18 | 268 | $10,433 | $18,217* | -$8,827 | -$151,483 |
| 21 | 375 | $14,626 | $18,385 | -$5,222 | -$170,897 |
| 24 | 501 | $19,552 | $18,582 | -$985 | **-$178,225** |
| 30 | 821 | $38,577 | $19,343 | +$15,377 | -$119,362 |
| 36 | 1,177 | $55,321 | $20,013 | +$29,776 | +$22,822 |

*Month 18: first hire (DevRel/Community) at $7,000/month.

**Bootstrap break-even: Month 25 (~$20K MRR).**
**Peak cumulative burn (runway required): ~$178,000**, reached at Month 24.

This is the single most important correction versus the prior model. The old plan claimed break-even at Month 9 on ~$66–95K of savings. That was an artifact of booking revenue in Month 2. On a realistic build timeline, the trough is roughly **$178K** and break-even is **~Month 25** — which is precisely why a seed round is the rational path rather than a stretch goal.

---

## Seed Round Scenario ($500K)

**Raise:** $500K pre-seed at $3M post-money (16.7% dilution).

The seed does two things the bootstrap cannot: covers the ~$178K trough with comfortable margin, and funds a DevRel hire at Month 9 (not 18) plus an infra engineer at Month 15. Earlier DevRel is the lever that moves the base case toward the seed-accelerated upside.

**Use of funds (18-month runway):**

| Category | Amount | % |
|---|---:|---:|
| Founder salary (18 × $10K) | $180,000 | 36% |
| DevRel/Community hire (from M9) | $120,000 | 24% |
| Infra engineer (from M15) | $60,000 | 12% |
| Infrastructure (AWS scaling) | $55,000 | 11% |
| GPU fleet (only if Phase-3 utilization justifies — see note) | $30,000 | 6% |
| Marketing + conference | $25,000 | 5% |
| Legal, accounting, ops | $30,000 | 6% |
| **Total** | **$500,000** | **100%** |

> GPU note: the $30K GPU line is gated on sustained utilization, not spent on schedule. Until the private fleet clears the utilization break-even in [gpu-serving-design.md](../06-design/gpu-serving-design.md), the private tier is served by **reselling hosted Gemma 4** ($0.33/1M output, commodity), which is cheaper than an under-utilized owned GPU. Build the fleet when volume — not the roadmap — says so.

**Seed-accelerated trajectory:** earlier acquisition support lifts gross adds roughly 1.5–1.7× from Month 10, putting Year-3 ARR near **$1.2M** and pulling break-even into the Month 14–16 range. This is the upside the seed buys — not a guarantee.

---

## 3-Year Revenue Projection

| Year | Ending MRR (base) | ARR (base) | Customers (base) | ARR (seed-accelerated) |
|---:|---:|---:|---:|---:|
| Year 1 | $3,349 | ~$40K | ~110 | ~$70K |
| Year 2 | $19,552 | ~$235K | ~500 | ~$450K |
| Year 3 | $55,321 | **~$664K** | ~1,180 | ~$1.2M |

Base case growth is *net of churn* throughout. The deceleration is built in: gross-add growth decays from ~13% to ~3% MoM over the three years, and churn removes 3.6–5.0% of the base every month. Any plan that shows smooth 20% net MoM for 36 months is ignoring the leak.

---

## Break-Even Analysis

| Scenario | Break-even MRR | Break-even month | Capital required to trough |
|---|---:|---:|---:|
| Bootstrap (hire at M18) | ~$20,000 | Month 25 | **~$178,000** |
| Bootstrap (no hire ever) | ~$12,000 | ~Month 19 | ~$135,000 |
| Seed ($500K) | ~$30,000 | Month 14–16 | Funded with margin |

**Practical plan:** the bootstrap-on-savings path now requires ~$135–178K, not $66K. Unless that capital is on hand, raise a small pre-seed/seed to clear the trough. The honest framing for investors is: *"$500K buys 18 months, clears a $178K trough, and funds the DevRel hire that converts the base case into the seed-accelerated case."*

---

## Sensitivity (the two variables that matter)

| Lever | Base | If worse | Year-3 ARR impact |
|---|---|---|---|
| Blended monthly churn | 4–5% | 7% (Builder-heavy, weak stickiness) | Year-3 ARR falls ~35% to ~$430K |
| Gross-add growth | decays 13%→3% | flat ~5% from M12 | Year-3 ARR falls ~30% to ~$465K |

Churn is the dominant risk because Builder ($29, 43% of mix) is the leakiest tier. This is why the validation gates now track DSL-stickiness directly — see [go-no-go-criteria.md](../04-validation/go-no-go-criteria.md).

---

## Funding Milestones for Investor Conversations

| Milestone | MRR | ARR | What it unlocks |
|---|---:|---:|---|
| Pre-seed / angels | $0 | $0 | Clears the build-phase + trough |
| Seed ($500K) | ~$10K (M18) | ~$120K | 18-month runway, DevRel + infra hire |
| Series A ($3M) | $80–120K | $1–1.5M | Enterprise motion, sales team |

---

## Related

- [Revenue Model](revenue-model.md) — unit economics and the bottoms-up the base case reconciles to
- [Go/No-Go Criteria](../04-validation/go-no-go-criteria.md) — revised gate timeline + DSL-stickiness gates
- [GPU Serving Design](../06-design/gpu-serving-design.md) — buy-vs-build utilization break-even for the GPU line
- [Executive Summary](executive-summary.md) — the ask, grounded in the ~$178K trough
