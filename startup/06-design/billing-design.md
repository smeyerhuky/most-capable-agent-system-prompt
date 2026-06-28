---
type: Design Doc
title: "AgentForge Billing Design"
description: Design for the AgentForge billing system. Covers Stripe metered billing, AWS Bedrock cost pass-through reconciliation, GPU tier credit metering, subscription tiers, and the usage dashboard.
tags: [design, billing, stripe, bedrock, pass-through, metering, subscription]
timestamp: 2026-06-28T00:00:00Z
---

# AgentForge Billing Design

---

## Billing Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    BILLING FLOW                             │
│                                                             │
│  LLM Call completes                                         │
│       │                                                     │
│       ▼                                                     │
│  Runtime emits LLMCallCompleted event                       │
│  { model, input_tokens, output_tokens, cost_usd }           │
│       │                                                     │
│       ▼                                                     │
│  Billing Service (async consumer):                          │
│  1. Record usage in Postgres (billing_events table)         │
│  2. Update run cost accumulator                             │
│  3. POST Stripe usage record (metered subscription)         │
│       │                                                     │
│       ▼                                                     │
│  Nightly reconciliation job:                                │
│  1. Fetch AWS Cost Explorer → actual Bedrock spend          │
│  2. Compare with Stripe usage records                       │
│  3. If delta > 1%: alert + freeze billing until resolved    │
└─────────────────────────────────────────────────────────────┘
```

---

## Subscription Tiers

| Tier | Price | Execution Limit | Bedrock Access | GPU Tier | Features |
|---|---:|---|---|---|---|
| Hobbyist | Free | 500 runs/month | No (community models only) | No | Public pipelines, basic dashboard |
| Builder | $29/month | 5,000 runs/month | Yes (pass-through) | 1M GPU tokens/month | Private pipelines, full dashboard, API access |
| Team | $99/seat/month | Unlimited | Yes (pass-through) | 5M GPU tokens/seat/month | Team workspaces, CI/CD, priority support |
| Enterprise | $2,000+/month | Unlimited | Yes (pass-through) | Dedicated GPU allocation | SSO, VPC deployment, SLA, audit logs |

A "run" = one `POST /runs` call. A single run may execute many LLM calls and tool calls — those are not metered as separate runs. Overage above limits: Builder users are gated (run blocked with an upgrade prompt); Team is unlimited.

---

## Stripe Integration

### Stripe Objects Per User

- **Customer:** Created on sign-up, with `email` and `metadata: { user_id: uuid }`
- **Subscription:** One subscription per user with:
  - A flat recurring price (e.g., $29/month for Builder)
  - A metered price for Bedrock token usage (priced at $0 — usage is reconciled separately and invoiced as a line item)
  - A metered price for GPU tokens (priced at $0.10/1M tokens over the included allowance)
- **Invoice:** Monthly, auto-collected. Includes: flat subscription fee + Bedrock pass-through charges + GPU overage.

### Bedrock Pass-Through Line Item

Bedrock tokens are NOT metered directly via Stripe's metered billing (too high frequency). Instead:

1. At month end, the reconciliation job totals actual Bedrock cost from AWS Cost Explorer (per user, via cost allocation tags)
2. A Stripe invoice item is created with the exact Bedrock cost as a one-time charge on the next invoice
3. A Stripe credit is issued for any difference if the estimate was already charged

This is the cleanest way to ensure zero markup — we invoice exactly what AWS charged.

### Cost Allocation Tags on Bedrock Calls

Every Bedrock API call is tagged with:
```
AgentForge-UserID: <user_id>
AgentForge-RunID: <run_id>
AgentForge-PipelineID: <pipeline_id>
```

AWS Cost Explorer allows filtering by these tags. The reconciliation job groups costs by `AgentForge-UserID` to get per-user spend.

---

## GPU Tier Metering

GPU token usage is metered directly in the billing database:

1. `GemmaVLLMAdapter.call()` returns token counts from vLLM response
2. Billing service records to `billing_events` table with `model_tier='gpu'`
3. GPU token usage is tracked against the plan's included allowance
4. Overage: metered as Stripe usage records at $0.10/1M tokens

GPU token cost to AgentForge: ~$0.05–0.10/1M tokens (amortized `g5.xlarge` spot cost). At $0.10/1M charged, margin is 0–50% depending on spot pricing. The GPU tier is not a profit center — it's a retention tool. Margin comes from the subscription.

---

## Execution Quota Enforcement

Run quota is enforced at `POST /runs` before execution begins:

```python
async def check_quota(user_id: str, plan: Plan) -> QuotaResult:
    key = f'quota:{user_id}:{current_month()}'
    count = await redis.incr(key)
    await redis.expire(key, seconds_until_month_end())
    
    if count > plan.run_limit:
        return QuotaResult(allowed=False, reason='monthly_limit_exceeded')
    return QuotaResult(allowed=True, remaining=plan.run_limit - count)
```

Redis INCR is atomic — no race condition on quota counting. The key expires at month end.

---

## Nightly Reconciliation Job

Runs at 02:00 UTC daily. Steps:

1. Fetch AWS Cost Explorer data for the previous day, grouped by `AgentForge-UserID` tag
2. For each user with Bedrock spend:
   a. Compare with `billing_events` records for the same day
   b. Calculate delta (reconciliation_error = |ce_cost - recorded_cost| / ce_cost)
   c. If delta > 1%: write to `billing_discrepancies` table, alert founder via email + PagerDuty
3. At month end: run final reconciliation, create Stripe invoice items for each user

**Discrepancy handling:** Do NOT auto-charge on discrepancies. Alert founder, investigate, then manually resolve before the invoice date. Billing accuracy is foundational trust.

---

## Usage Dashboard (Customer-Facing)

Available via `GET /billing/usage`. The web IDE billing page shows:

```
Month: June 2026
Plan: Builder ($29/month)

Subscription:          $29.00
Bedrock usage:         18,240,000 tokens   = $8.47
  Claude 3.5 Sonnet:  12,100,000 tokens   = $7.26
  Llama 3 70B:         6,140,000 tokens   = $1.21
GPU tier usage:        2,200,000 tokens    = $0.00 (within 1M included)
  Gemma 4-26B-A4B:    2,200,000 tokens

Runs executed:         342 / 5,000
Estimated total:       $37.47

Detailed breakdown by pipeline:
  RefactorPipeline:   156 runs, 9.2M tokens, $4.23
  TestGenerator:       98 runs, 5.8M tokens, $2.89
  DocWriter:           88 runs, 3.2M tokens, $1.35
```

This transparency is a product differentiator — users can see exactly what they're spending and why.

---

## Postgres Schema (Billing Tables)

```sql
-- Records every metered event (LLM calls, tool calls with costs)
CREATE TABLE billing_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    run_id UUID NOT NULL REFERENCES runs(id),
    user_id UUID NOT NULL REFERENCES users(id),
    event_type VARCHAR(50) NOT NULL,  -- 'llm_call', 'gpu_call'
    model VARCHAR(100) NOT NULL,
    model_tier VARCHAR(20) NOT NULL,  -- 'bedrock', 'openai', 'gpu'
    input_tokens INT NOT NULL,
    output_tokens INT NOT NULL,
    cost_usd DECIMAL(10, 6) NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Monthly rollup for fast dashboard queries
CREATE TABLE billing_monthly_summary (
    user_id UUID NOT NULL REFERENCES users(id),
    month DATE NOT NULL,  -- first day of month
    bedrock_cost_usd DECIMAL(10, 4),
    gpu_tokens BIGINT,
    gpu_cost_usd DECIMAL(10, 4),
    runs_count INT,
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (user_id, month)
);

-- Discrepancy log for reconciliation audit trail
CREATE TABLE billing_discrepancies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    period DATE NOT NULL,
    recorded_cost_usd DECIMAL(10, 4),
    aws_ce_cost_usd DECIMAL(10, 4),
    delta_pct DECIMAL(6, 2),
    resolved BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

---

## Related

- [Data Model](data-model.md) — full schema context
- [API Spec: Billing Endpoints](../05-specs/api-spec.md#6-billing)
- [Revenue Model](../07-business-plan/revenue-model.md) — pricing rationale and unit economics
- [GPU Serving Design](gpu-serving-design.md) — GPU cost basis for metering
