---
type: Business Plan Section
title: "AgentForge Sales Playbook — Sole Founder Edition"
description: Month-by-month sales guide for a sole technical founder. Covers the Collison installation method for first 10 customers, outbound scripts, HN/GitHub launch, demo-only motion, and when to hire the first salesperson. Grounded in YC best practices.
tags: [business-plan, sales, sole-founder, yc, collison-installation, outbound, demo, playbook]
timestamp: 2026-06-28T00:00:00Z
---

# AgentForge Sales Playbook — Sole Founder Edition

> "You are the only person in the company who can sell your product right now. Not because you're the best salesperson — but because you understand what you built well enough to tell the exact right customer why it's exactly right for them." — YC Dessaigne

---

## The Core Principle

Do not build a sales process before you understand why people buy. The first 10 customers are not a sales problem — they are a customer development opportunity. Every conversation is research. Every "no" tells you what to build next.

Do not have a sales deck before $10M ARR. Run demos only.

---

## Month 1: The Collison Installation (First 10 Customers)

The "Collison installation" (named after Stripe's founder): go to your customers physically (or digitally), install your product for them, and watch them use it. Do not ask them to self-serve. Do not send them a signup link. Do it for them.

### Who to Target

10 specific humans in your existing network who:
- Are engineers at companies building with LLMs (look for LangChain/LangGraph in their GitHub)
- Are people you can reach with one message (LinkedIn, Twitter DM, email, Slack)
- Are at companies with 10–200 engineers (small enough to move fast, large enough to have a real workflow problem)

### The Outreach Script

**Email / LinkedIn DM:**
> "Hey [Name] — I've been building a programming language for LLM agent workflows — essentially a DSL that compiles multi-step pipelines down to structured API calls, with a type checker that catches errors before runtime. I'm looking for 5 engineers who are actively building this kind of thing to try it and give me feedback. No pitch, no commitment. I'd set it up for you and watch you use it. Would you be up for a 45-minute video call this week?
>
> [Founder Name]"

**What you're NOT doing:** Asking them to sign up. Asking them to pay. Sending a one-page PDF. Pitching investors.

**What you ARE doing:** Asking for their time in exchange for solving their problem — in person.

### The Installation Call (45 minutes)

1. **5 min:** "Tell me about the most painful agent workflow you've built recently."
2. **10 min:** Ask them to share screen. Watch them try to build what they just described in AgentForge. Say nothing — watch where they get confused.
3. **15 min:** You build it with them, narrating what the compiler is doing and why.
4. **10 min:** "What would make this 10× more useful for you?"
5. **5 min:** "Would you pay $29/month for this if the [blocker they named] was fixed? Who else on your team should try this?"

### After the Call

- Fix the top blocker they named within 48 hours
- Email them: "Fixed the thing you mentioned. Can I show you the update next week?"
- Ask for one warm introduction: "Who else on your team or in your network builds multi-step agent pipelines?"

### Target: 10 installation calls → 3–5 paying customers

A 30–50% conversion rate from installation call to paid is realistic when you're solving a real pain and delivering personal setup. Do not worry about the other 5–7 — worry about why they didn't convert.

---

## Month 2–3: Expand to First 100 via Network + Content

### Warm Network Outreach (50 engineers)

Expand the Collison installation to 50 more engineers via warm introductions from your first 10 customers. Each customer should be able to introduce you to 2–5 people.

**Introduction script to send your customers:**
> "If you know any other engineers who are building multi-step LLM agent pipelines — especially if they're frustrated with LangGraph or managing context windows — I'd love an introduction. I'll set it up personally for them. It takes 45 minutes."

### LinkedIn Content (Weeks 5–8)

3–4 posts per week. Topics that work for a technical audience:
- "I built a compiler for LLM agent workflows. Here's how the type system works." (with code)
- "The problem with LangGraph isn't the library — it's the abstraction level." (contrarian, technical)
- "Refactored a 400-line LangGraph script to 30 lines of `.agent`. Here's the diff." (show, don't tell)
- "Why I believe agent workflows should be compiled, not interpreted." (philosophy)

Each post ends with: "Building AgentForge in public. DM me if you want early access."

### Twitter/X Thread (Week 6)

A 12-part thread explaining the full compiler pipeline for agent workflows, ending with a link to the GitHub repo. Target: 500 impressions minimum from technical audience.

---

## Month 4–6: Show HN + GitHub Launch

**The most important distribution event of the company's first year.**

### Show HN Post (Phase 2 — see [customer-acquisition.md](customer-acquisition.md) for the template)

Goal: ≥ 200 upvotes, ≥ 50 comments, ≥ 100 GitHub stars in 24 hours.

**During the HN launch (4-hour window):**
- Respond to every comment within 10 minutes
- When someone asks a hard question: "Great question. Here's why we made that call..." (then explain the design decision)
- When someone says "why not just use LangGraph": give the prepared answer (see [talking-points.md](talking-points.md))
- Do not be defensive. Be curious. Every comment is feedback.

### GitHub Launch

- README with a 5-minute quickstart that gets a developer to a working pipeline in under 5 minutes
- A GIF in the README showing compile + run from the VS Code extension
- "Show HN" style: show the working thing, not the concept

---

## Month 7+: Outbound for Enterprise Leads

Once you have ≥ $2,000 MRR and clear product-market fit signals (users who use AgentForge 3+ times/week), begin targeted enterprise outreach.

### Identify Champions

Target: engineering managers and staff engineers at 50–500 person companies who are actively building LLM-powered products.

Sources:
- LinkedIn: search "staff engineer" + "machine learning" + "workflow" at companies known to use LLMs
- GitHub: find engineers who have public LangGraph or LangChain projects → they have the exact pain
- Twitter/X: engineers who tweet about "building agents" or "prompt engineering at scale"

### Outbound Email Template (Cold)

Subject line: "Multi-step LLM workflows at [Company]"

> Hi [Name],
>
> I noticed your team is building with [LangGraph/LangChain/Anthropic API] — I've been working on a tool that might be useful.
>
> AgentForge is a DSL for multi-step LLM agent workflows. The compiler catches type mismatches between pipeline steps before any API call fires — which means no more "the context window was wrong at step 4" debugging. Model routing (Claude, GPT-4o, or a private Gemma 4 GPU) is one line change.
>
> I'd love to show you a 15-minute demo. No slides — just a live `.agent` file being compiled and run. Worth a look?
>
> [Founder Name]

**What makes this work:**
- Specific (mentions their stack)
- Shows knowledge of their pain
- Asks for a demo, not a meeting
- Short (under 150 words)

### Demo Script (15 minutes, no slides)

1. Open VS Code with a blank `.agent` file
2. Type the refactor pipeline from scratch, narrating what each line does
3. Intentionally introduce a type error — show the compile error in the IDE
4. Fix it — show the green checkmark
5. Hit "Run" — show the live dashboard
6. Ask: "How does this compare to how you currently build this kind of pipeline?"
7. Close: "If this would save [problem they described], would you want to try it?"

---

## When to Hire the First Salesperson

**Not before $1M ARR.** At $1M ARR, you understand why people buy. You have a repeatable process (the demo works, the outbound email has a measurable response rate, the HN launch playbook is documented). Only then can you hand this to someone else.

**First sales hire criteria (at ~$1M ARR):**
- Has sold technical developer tools before (not SaaS to sales ops — developer tools)
- Can pass your coding challenge (they need to be able to demo the product credibly)
- Has 3+ references from engineers (not just managers) who liked working with them
- Compensation: 60% base, 40% commission on new ARR
- First metric: 30-day target of 5 qualified demos booked from cold outbound

---

## Related

- [Talking Points](talking-points.md) — prepared answers for common objections
- [Customer Acquisition](customer-acquisition.md) — tactical week-by-week first 1,000 users
- [Team and Hiring](team-and-hiring.md) — when and how to hire the first salesperson
