---
type: Validation
title: "AgentForge Pre-Build Validation Experiments"
description: Five validation experiments to run before writing production code. Each experiment tests a key business or technical assumption and has a clear pass/fail criterion. Results feed directly into Phase 1 go/no-go decision.
tags: [validation, experiments, pre-build, assumptions, customer-discovery]
timestamp: 2026-06-28T00:00:00Z
---

# AgentForge Pre-Build Validation Experiments

Run these experiments before writing any production code. Each tests a core assumption. If an assumption fails, pivot the approach before investing weeks of engineering.

**Time budget:** 2 weeks total. These can overlap.

---

## Experiment 1: Customer Pain Validation

**Assumption being tested:** Senior developers building multi-step LLM workflows experience real, acute pain around portability, auditability, and compile-time correctness — and they will pay to solve it.

**Method:**
1. Identify 20 potential customers: engineers at companies known to be building with LLMs (look for job postings mentioning "LangChain", "LangGraph", "agent workflow", "prompt engineering"). LinkedIn, Twitter/X, and personal network.
2. Send a 2-sentence cold message: *"I'm building a developer tool for LLM agent workflows. Would you spend 20 minutes with me talking about how you currently build and debug multi-step agent pipelines? No pitch — just learning."*
3. Conduct 10 conversations (expect 50% response rate from warm network, 20% from cold).
4. Ask: What breaks most often in your agent workflows? How do you debug them? What would you pay for a tool that caught errors before runtime?

**Pass criterion:** ≥ 7 of 10 interviewees independently describe the portability/auditability/correctness problem without being prompted. ≥ 5 say they would pay $29+/month.

**Fail criterion:** < 5 describe the pain clearly, OR the pain they describe is different from what AgentForge solves (e.g., "I just want a better prompt editor" — signals wrong abstraction level).

**What to do if it fails:** Do not build. Pivot to whatever pain they actually described.

**Estimated time:** 1 week (outreach + interviews)

---

## Experiment 2: Pricing and Willingness to Pay

**Assumption being tested:** $29/month Builder tier is the right price. Free tier with 500 executions is a sufficient hook without cannibalizing paid conversion.

**Method:**
1. Show 20 developers (can overlap with Experiment 1) a 1-page concept description of AgentForge (3 bullet points, no pricing).
2. Ask: "What would you pay for this per month?" (open-ended, no anchoring)
3. Then ask: "Would you pay $29/month?" → "Would you pay $49/month?" → "Would you pay $99/month?"
4. Separately ask: "What free tier would make you try it without a credit card?" and "At what free tier limit would you upgrade to paid?"

**Pass criterion:** Median willingness to pay ≥ $29/month. ≥ 60% say they would try a free tier with 500 executions. ≥ 40% say they would upgrade to paid if they hit the free limit.

**Fail criterion:** Median WTP < $20/month (consider lower entry price or more generous free tier). OR < 20% say they'd upgrade (free tier too generous or product not compelling enough).

**What to do if it fails:** Adjust pricing model. If WTP is low, consider open-source-only with a cloud hosting upsell (similar to n8n's fair-code model). If upgrade intent is low, reduce free tier to 100 executions.

**Estimated time:** Overlaps with Experiment 1 (add pricing questions to same conversations)

---

## Experiment 3: Technical Feasibility — Compiler Prototype

**Assumption being tested:** The `.agent` DSL grammar is learnable, the compiler can be built by one engineer in 2 weeks, and the compiled IR correctly represents agent workflows.

**Method:**
1. Build a minimal prototype in 5 days: lexer, parser, and IR emitter only (no type checker, no runtime).
2. Write 5 sample `.agent` files covering: simple single-step, multi-step pipeline, loop, branch, and a pipeline with a back-edge.
3. Run all 5 through the prototype compiler. Measure: parse errors on valid input (should be zero), output IR correctness (manually verify against expected output).
4. Show the sample `.agent` files to 5 developers from Experiment 1. Ask: "Is this syntax learnable? What would you change?"

**Pass criterion:** Zero parse errors on all 5 valid sample files. IR output correctly represents all step types. ≥ 4 of 5 developers say the syntax is learnable (rate ≥ 4/5). Compiler prototype built in ≤ 5 working days.

**Fail criterion:** Parse errors on valid input (grammar has ambiguities — fix before proceeding). Developer feedback: syntax is confusing or too verbose (redesign grammar). Prototype takes > 10 days (scope was underestimated — plan for 4-week Phase 1A).

**What to do if it fails:** Spend 1 week revising the grammar based on developer feedback, then re-test. A DSL that developers hate will fail regardless of the runtime quality.

**Estimated time:** 1 week

---

## Experiment 4: Bedrock Integration Feasibility

**Assumption being tested:** AWS Bedrock pass-through billing can be implemented accurately, and the pass-through model (zero markup) is commercially viable at the expected volume.

**Method:**
1. Create an AWS account with Bedrock access (Claude 3.5 Sonnet via Bedrock).
2. Build the minimal Bedrock adapter: `POST /v1/messages` call, streaming response, token counting.
3. Run 100 test LLM calls with known inputs, record Bedrock costs via Cost Explorer.
4. Implement Stripe metered billing in test mode. Post usage records after each call. Reconcile: Stripe charges == Bedrock costs (within 1%).
5. Calculate: at 5,000 executions/month (Builder tier), what is the average Bedrock cost per execution? Is the $29/month subscription sustainable if average cost per execution is $X?

**Pass criterion:** Billing reconciliation error < 1%. Bedrock cost per typical execution ≤ $0.005 (i.e., $29/month covers ≥ 5,000 executions for a typical workflow). Streaming responses work correctly.

**Fail criterion:** Billing reconciliation error > 1% (unacceptable — fix before launch). Bedrock cost per execution > $0.010 (means subscription revenue doesn't cover costs at scale — raise prices or reduce included executions).

**What to do if it fails:** Billing mismatch: fix the reconciliation logic, do not launch with inaccurate billing. Cost too high: reduce the included execution count in the Builder tier, or add a cost cap per execution (auto-upgrade to pay-as-you-go when exceeded).

**Estimated time:** 3 days

---

## Experiment 5: GPU Tier Performance Validation

**Assumption being tested:** Gemma 4-26B-A4B with AWQ quantization and MTP speculative decoding achieves ≥ 50 tokens/sec on a `g5.xlarge` (A10G 24GB), making it a viable cost-competitive private tier.

**Method:**
1. Provision a `g5.xlarge` spot instance on AWS (or use a GPU rental — ~$0.35/hr spot).
2. Install vLLM. Load `google/gemma-4-26B-A4B-it` with AWQ quantization and the `-assistant` drafter.
3. Run 50 inference calls with representative prompts (code analysis, 2,000–4,000 token context, 500–1,000 token output).
4. Measure: tokens/sec (output), time-to-first-token (TTFT), MTP acceptance rate, VRAM usage, failure rate on spot preemption recovery.

**Pass criterion:** Tokens/sec ≥ 50 (output). TTFT ≤ 500ms. MTP acceptance rate ≥ 60%. VRAM usage ≤ 22GB (leaving 2GB headroom on 24GB A10G). Spot recovery < 60 seconds.

**Fail criterion:** Tokens/sec < 30 (not fast enough to be a competitive advantage). VRAM usage > 24GB (doesn't fit — switch to `g5.2xlarge` with 48GB, raises cost). MTP acceptance rate < 40% (speculative decoding not helping — investigate drafter configuration).

**What to do if it fails:** If VRAM is the issue: use 12B model instead of 26B-A4B (lower quality but fits easily). If tokens/sec is too slow: try `g5.2xlarge` or run multiple model replicas for throughput. If MTP doesn't work: ship without speculative decoding (still fast at 26B-A4B Q4 without MTP — just less fast).

**Estimated time:** 2 days

---

## Experiment Results Template

Record results here when experiments are complete. Then make a GO/NO-GO decision.

| Experiment | Status | Pass? | Key Finding | Decision |
|---|---|---|---|---|
| 1. Customer Pain | Pending | — | — | — |
| 2. Pricing WTP | Pending | — | — | — |
| 3. Compiler Prototype | Pending | — | — | — |
| 4. Bedrock Billing | Pending | — | — | — |
| 5. GPU Performance | Pending | — | — | — |

**Overall pre-build gate:** All 5 must pass (or have a clear remediation committed) before beginning Phase 1 production engineering.

---

## Related

- [Go/No-Go Criteria](go-no-go-criteria.md) — phase gates that use experiment results
- [Internal FAQ](../01-prfaq/internal-faq.md) — the assumptions these experiments test
- [Customer Acquisition](../07-business-plan/customer-acquisition.md) — Experiment 1 participants become first customers
