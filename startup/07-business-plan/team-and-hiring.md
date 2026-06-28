---
type: Business Plan Section
title: "AgentForge Team and Hiring Plan"
description: Sole founder strengths and gap analysis, hiring sequence (first through fifth hires), advisory board recruitment plan, and key-person risk mitigation strategy for AgentForge.
tags: [business-plan, team, hiring, founder, advisory-board, key-person-risk]
timestamp: 2026-06-28T00:00:00Z
---

# AgentForge Team and Hiring Plan

---

## Founding Team

**[Founder Name] — Sole Technical Founder**

**Experience (30 years):**
- Software architecture: designed and built distributed systems at scale
- Developer experience (DX): built developer tooling, CLIs, IDE extensions, SDKs
- Infrastructure: Kubernetes, AWS, multi-region deployments, GPU serving
- Testing: TDD practitioner, built CI/CD pipelines and test frameworks
- Compiler theory: academic and practical background in PL design

**Why this matters for AgentForge:**

| Domain | What's Needed | Founder Has It? |
|---|---|---|
| DSL grammar design | PL theory, EBNF, parser construction | Yes |
| Compiler implementation | Lexer/parser/AST/IR, type systems | Yes |
| Cloud infrastructure | AWS, Kubernetes, ECS, RDS, Redis | Yes |
| Developer experience | VS Code extensions, CLI design, SDK ergonomics | Yes |
| GPU serving | vLLM, CUDA, spot instances, speculative decoding | Yes (learning) |
| Sales | Technical demos, customer conversations, outbound | Partially |
| Design / UX | Web IDE visual design | No — hire/contract |
| Enterprise sales | Deal structuring, legal, procurement | No — hire after $1M ARR |
| Marketing / content | Blog, SEO, social media | Partially — can do technical content |

**Honest assessment:** The founder can build the full product and sell it personally for the first 12–18 months. The critical gap is enterprise sales (hire at $1M ARR) and design (contract in Phase 1).

---

## Key-Person Risk

This is the primary risk of a sole founder. Mitigation plan:

### Documentation as Insurance

The current document bundle ("Startup in a Box") is the first line of defense. A new engineering hire could onboard to AgentForge in 1 week using:
- [RFC-001](../03-rfc/rfc-001-agent-harness.md) — architecture
- [Phase 1 PRD](../02-prd/phase-1-mvp.md) — task decomposition
- [Language Spec](../05-specs/language-spec.md) + [Runtime Spec](../05-specs/runtime-spec.md) — formal specs
- [Compiler Design](../06-design/compiler-design.md) — implementation guide

### Open-Source Community

The compiler being MIT open-source means: if the founder is incapacitated, the community can maintain the core language toolchain independently. The cloud runtime requires more knowledge transfer but is not the language's value — the language is.

### Advisory Board (Target: recruit by Month 4)

3 advisors, each with a specific domain the founder doesn't cover deeply:

**Advisor 1: Enterprise Sales / GTM**
- Profile: has led sales at a developer tools SaaS from $0 to $5M ARR
- Commitment: 2 hours/month, advisor equity (0.25%)
- What they provide: enterprise deal structuring, prospect introductions, pricing feedback

**Advisor 2: LLM Infrastructure / ML Engineering**
- Profile: senior engineer who has run LLM inference at scale (GPU fleet, quantization, serving)
- Commitment: 2 hours/month, advisor equity (0.25%)
- What they provide: GPU serving decisions, model evaluation, vLLM troubleshooting

**Advisor 3: Developer Tools / PLG**
- Profile: has launched and grown a developer tool product (ideally with a PLG motion)
- Commitment: 2 hours/month, advisor equity (0.25%)
- What they provide: PLG mechanics, HN/GitHub launch tactics, VS Code Marketplace growth

**Total advisor equity: 0.75%** — standard and appropriate for this stage.

---

## Hiring Sequence

> **Timeline note:** dates here are re-baselined to match the churn-netted financial model (see [financial-model.md](financial-model.md)). The old schedule (DevRel Month 7 / Infra Month 10) assumed revenue from Month 2, which is not achievable. Bootstrap and seed timelines diverge significantly — both are shown where they differ.

### Hire #1: DevRel / Community Manager

**Timing:** Month 18 (bootstrap, ~$10K MRR) — Month 9 (seed-funded, earlier acquisition support is the primary lever the seed buys)

**Why this hire first:**
- Phase 2 growth depends on community building (Discord, GitHub, blog, Twitter)
- The founder can't write code, manage customer conversations, AND do content simultaneously
- DevRel is a force multiplier on PLG — one good DevRel person driving 100 GitHub stars is worth more than a junior engineer

**Profile:**
- Has built a developer community before (Discord, GitHub community, or similar)
- Can write technical content (blog posts, cookbook articles) at engineer-level quality
- Bonus: has experience as a developer, can demo AgentForge credibly

**Compensation:** $80K–$100K base + 0.5% equity (seed-era standard)

**First 30-day targets:**
- Publish 2 technical blog posts
- Grow Discord to 200 members
- Get 100 new GitHub stars
- Interview 5 users and write a customer story

### Hire #2: Infrastructure / Reliability Engineer

**Timing:** Month 15 (seed-funded, ~$7K MRR) — post-break-even bootstrap (~Month 28–30, ~$35K MRR)

**Why this hire second:**
- GPU fleet management (Phase 3) requires dedicated ops attention
- As users scale, reliability work (monitoring, incident response, database tuning) requires a dedicated person
- Frees the founder to focus on product and sales

**Profile:**
- Experienced with AWS, Kubernetes, PostgreSQL at scale
- Has run GPU infrastructure (bonus)
- Comfortable with Python async/microservices

**Compensation:** $130K–$150K base + 0.5% equity

### Hire #3: Full-Stack Engineer

**Timing:** post-break-even, when MRR sustains ~$40K (bootstrap ~Month 32–36; earlier with seed)

**Why this hire third:**
- Web IDE features (Phase 2–3) need dedicated engineering bandwidth
- The founder's time is increasingly valuable in sales conversations — engineering needs to scale independently

**Profile:**
- Full-stack (React + Python/FastAPI)
- Has built developer-facing products (VS Code extensions, web IDEs, or similar)
- Product-minded: can scope features and write acceptance criteria

**Compensation:** $140K–$160K base + 0.5% equity

### Hire #4: First Sales / Account Executive

**Timing:** at $1M ARR (the MRR trigger, not a fixed month — on the bootstrap path this is ~Year 4; seed-accelerated pulls it closer to Year 3)

**Why this hire at $1M ARR:**
- YC Dessaigne: "Hire your first salesperson at $1M ARR. Not before."
- Before $1M ARR, the founder understands why people buy better than any hired salesperson can
- At $1M ARR, the sales motion is clear enough to hand off

**Profile:**
- Has sold developer tools (not SaaS to operations — developer tools to engineers)
- Can pass a technical interview (must understand and demo AgentForge credibly)
- Has 3+ references from engineers (not just managers) who valued working with them
- **Explicitly does NOT need to code well** — needs to understand code

**Compensation:** 60% base ($90K), 40% commission on new ARR. OTE: $150K at plan.

### Hire #5: Designer (Contract → FTE, Phase 3)

**Why a designer:**
- The web IDE's visual quality is a trust signal for enterprise customers
- Phase 3 dashboard (GPU metrics, MTP stats, cost breakdown) is data-dense and needs UX care
- The landing page can be improved by a designer in 2 weeks more than a founder in 2 months

**Start as:** Contract (project-based, ~$150/hr, 40–60 hours for landing page + dashboard redesign)

**Convert to FTE** if design is a sustained bottleneck (> 20 hours/week of design work needed consistently).

---

## Organizational Principles

**Phase 1 (Months 1–18 bootstrap / 1–9 seed, 1 person):** Founder does everything. Ship fast, stay focused.

**Phase 2 (post-DevRel hire, 2 people):** Founder (product + sales) + DevRel (community + content). Weekly 1:1, shared OKRs.

**Phase 3 (post-infra hire, 3–4 people):** Founder (product + strategic sales) + DevRel + Infra Engineer (+ Full-Stack when MRR sustains it). Weekly team standup, monthly planning.

**At $1M ARR (5 people):** Add first AE. Founder spends 50% on product, 30% on strategic sales (enterprise), 20% on management.

**Guiding hiring principles:**
- Hire for domain depth over general competence
- Every hire must be able to answer: "What will you have demonstrably done in 90 days?"
- No hire without reference calls with 3 people they've worked with (not managed — worked with)
- Equity is meaningful at this stage — use it to attract people who believe in the mission

---

## Related

- [Financial Model](financial-model.md) — salaries and timing embedded in the P&L
- [Sales Playbook](sales-playbook.md) — criteria for Hire #4 (first AE)
- [Executive Summary](executive-summary.md) — team section for investors
