---
type: Business Plan Section
title: "AgentForge Customer Acquisition — First 1,000 Users"
description: Tactical, week-by-week guide for acquiring the first 1,000 users. Covers first 10 (Collison installation), first 100 (warm network + content), first 1,000 (HN + GitHub launch + Discord), and first enterprise customer (bottom-up champion playbook).
tags: [business-plan, customer-acquisition, growth, hn, github, discord, enterprise, plg]
timestamp: 2026-06-28T00:00:00Z
---

# AgentForge Customer Acquisition — First 1,000 Users

---

## First 10 Users (Weeks 1–4)

**Method:** Collison installation (see [sales-playbook.md](sales-playbook.md) for scripts)

**Who:** Personal network — engineers you know who build multi-step LLM workflows.

**Goal:** Not conversion. Understanding. Why do they love it? Why do they hesitate? What would make them pay immediately?

**Metrics to track:**
- Time-to-first-run during installation call
- First question they ask (signals what the README should answer)
- First point of confusion in the editor (signals what Intellisense should autocomplete)
- Did they ask to come back? (strongest signal)

**Outcome:** 3–5 paying users, 10 deep product insights, 10 warm intros to their networks.

---

## First 100 Users (Weeks 5–10)

### Warm Network Expansion

Each of your first 10 customers introduces you to 3–5 engineers. Offer to do the installation personally for all of them.

**Conversion target:** 50 installation calls → 15–20 paying users.

### LinkedIn Content Series (Weeks 5–10)

A 6-week series, 3 posts per week:

**Week 5: Problem Statement**
- Post 1: "Why building production LLM agent pipelines is still broken in 2026." (shares the debugging-at-step-7 story)
- Post 2: Code comparison post: 400 lines of LangGraph vs. 30 lines of `.agent`. Actual diff, no commentary needed.
- Post 3: "The 3 problems no current LLM framework solves." (portability, auditability, compile-time correctness)

**Week 6: The Solution**
- Post 4: "How we built a type system for LLM tool calls." (technical deep-dive, attracts engineers)
- Post 5: Demo GIF of the VS Code extension — type error, fix, green checkmark, run, dashboard. No words needed.
- Post 6: "The `.agent` file format: a 5-minute introduction." (links to GitHub)

**Week 7: Social Proof**
- Post 7: Beta user quote (ask your first 5 paying users for a sentence about why they use it)
- Post 8: "What I learned from 50 engineering conversations about LLM orchestration." (synthesis of customer discovery)
- Post 9: "The template that saved our team 3 days of debugging." (a real story from a beta user)

**Week 8–10:** Continue with technical deep-dives, cookbook content, and community highlights.

### Twitter/X Thread (Week 7)

12-part thread: "I built a compiler for LLM agent workflows. Here's the complete design."

Structure:
1. Hook: "Building multi-step LLM pipelines in 2026 is still writing assembly. Here's what a compiler looks like for AI agents. 🧵"
2–4: The problem (with code examples)
5–8: The solution (grammar → parser → AST → IR → runtime, each with one example)
9–10: The type system (the "this is why it matters" moment)
11: Link to GitHub
12: "Give it a try and tell me what's broken."

**Goal:** 5K+ impressions, 100+ GitHub stars from the thread alone.

---

## First 1,000 Users (Weeks 11–20)

### Show HN Post (Week 12–13, after VS Code extension ships)

**Timing:** Tuesday or Wednesday at 9am EST. Do not post on Mondays (oversaturated) or Fridays (low engagement).

**Title (test variants):**
- "Show HN: AgentForge – A programming language that compiles to LLM agent workflows"
- "Show HN: I built a type checker for multi-step LLM pipelines"
- "Show HN: AgentForge – Write .agent files, compile them, run on Claude/GPT-4o/private GPU"

**Post body template:**
> Hi HN,
>
> I've been building AgentForge — a DSL compiler and cloud runtime for multi-step LLM agent workflows.
>
> The problem: building reliable agent pipelines today means writing Python orchestration glue with no compile-time validation. When step 7 fails because step 3 produced the wrong type, you find out at runtime. When you want to switch from Claude to GPT-4o, you rewrite orchestration code.
>
> The solution: `.agent` files. Here's a 30-line pipeline that finds dead code in a TypeScript codebase, gets user approval, patches each file, and re-runs tests until they pass:
>
> [code block — the refactor pipeline from the press release]
>
> The compiler type-checks tool contracts before any API call fires. The runtime routes to Claude (via AWS Bedrock), GPT-4o, or a private Gemma 4-26B GPU. The dashboard shows live token spend, stage progress, and model latency.
>
> The compiler is MIT open-source: [GitHub link]. The cloud runtime is $29/month (Builder).
>
> What I'm most curious to hear: Is the DSL the right abstraction, or is Python + library the right answer? Happy to defend either side.
>
> [Founder Name]

**During the 4-hour HN window:**
- Be in front of your computer the entire time
- Reply to every comment within 10 minutes
- The question "why not just use LangGraph?" will be asked 5 times. Answer it differently each time (different angles of the same answer).
- Do not be defensive. Be curious. Thank people for hard questions.

**Goal:** 200+ upvotes, 50+ comments, 300+ GitHub stars in 24 hours, 100+ sign-ups.

### GitHub Launch Checklist

Before the HN post:
- [ ] README with a 5-minute quickstart (copy-paste-run in < 5 minutes)
- [ ] GIF in README showing VS Code Intellisense + compile + run
- [ ] 10 sample `.agent` files in `/examples`
- [ ] `CONTRIBUTING.md` with "good first issue" guidelines
- [ ] 5 "good first issue" issues filed (small, well-defined, with acceptance criteria)
- [ ] GitHub Discussions enabled
- [ ] GitHub Sponsors enabled (some users will want to support even before paid)
- [ ] `CHANGELOG.md` with v0.1.0 entry

### Product Hunt Launch (Week 16)

**Timing:** After HN launch. Use HN momentum (GitHub stars, testimonials) as social proof.

**Post type:** Product of the Day.

**Hunter:** Ask your most technically credible beta user to hunt it (not yourself — hunted-by matters on PH).

**Goal:** Top 5 Products of the Day → 2K+ upvotes → 200+ sign-ups.

### Discord Community (ongoing from Week 10)

Create an `agentforge` Discord server before the HN launch. Structure:
- `#general` — community chat
- `#show-your-pipelines` — users share their `.agent` files (virality engine)
- `#bug-reports` — faster than GitHub issues for early users
- `#feature-requests` — structured feedback
- `#cookbook` — copy-paste patterns for common tasks

**Community rules:**
- Founder responds to every message in the first 90 days (sets the tone for quality)
- No tolerance for off-topic content (keep it developer-focused)
- Monthly "Pipeline of the Month" feature → blog post → social proof

**Discord as acquisition:** `#show-your-pipelines` posts are public-facing content. The best ones become "Cookbook" blog posts that drive organic search traffic.

### Blog + SEO (Weeks 14+)

5 initial blog posts targeting organic search:

1. "How to build a multi-step LLM agent workflow that doesn't break at 3am" (target: "LLM agent workflow" — 5K searches/month)
2. "LangGraph vs. AgentForge: which is right for your team?" (target: "LangGraph alternative")
3. "Gemma 4 MTP speculative decoding: 2x faster inference on a $0.35/hr GPU" (target: "Gemma 4 vLLM")
4. "The `.agent` file format: a complete guide" (target: "agent workflow DSL")
5. "Building a code refactor agent that actually works: a case study" (target: "AI code refactor agent")

---

## First Enterprise Customer (Months 5–8)

### Identify Champions

The first enterprise customer won't come from sales — it will come from a developer who uses AgentForge personally and then advocates for it at their company. The job is to find and support those champions.

**Champion identification signals:**
- Uses AgentForge > 10 times in a month
- Has invited team members to their workspace
- Has filed feature requests for enterprise features (SSO, audit logs)
- Has mentioned AgentForge positively on LinkedIn or Twitter

**Champion nurture:**
1. Email them personally: "I noticed you've been using AgentForge a lot — is there anything blocking you from using it with your whole team?"
2. Offer to do a free team demo and setup
3. Ask: "What would your company need to put this on a formal contract?" (compliance? SSO? VPC? audit logs?)
4. Build those things. This is how enterprise features get prioritized.

### The Enterprise Pilot

**Structure:** 90-day pilot. $0 cost to them. You get:
- A named case study ("Company X reduced agent pipeline debugging time by Y%")
- A reference customer for subsequent enterprise sales
- Detailed feedback on enterprise requirements

**Pilot requirements from them:**
- Designated primary contact (engineering manager or staff engineer)
- 2 bi-weekly check-in calls
- Written testimonial at pilot end (if they're happy)

**Conversion path:** Pilot → 90 days → if they're happy → $2,000/month Enterprise contract.

---

## Related

- [Sales Playbook](sales-playbook.md) — the scripts and templates for individual sales conversations
- [Go-to-Market](go-to-market.md) — channel strategy and partnership sequencing
- [Talking Points](talking-points.md) — prepared answers for conversations at each stage
