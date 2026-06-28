---
type: PRD Phase
title: "Phase 1 — MVP"
description: MVP phase requirements and agentic PDLC task decomposition for AgentForge. Covers the compiler core, Bedrock integration, web IDE, and pass-through billing. Target: 10 paying users in 8 weeks.
tags: [prd, mvp, phase-1, tasks, compiler, bedrock, billing]
timestamp: 2026-06-28T00:00:00Z
---

# Phase 1 — MVP

**Duration:** Months 1–2 (8 weeks)
**Gate:** [Phase 1 Go/No-Go](../04-validation/go-no-go-criteria.md#phase-1-gate)
**Goal:** 10 paying users running `.agent` files in production on AWS Bedrock, with real pass-through billing and a functional web IDE.

---

## What Ships in Phase 1

1. AgentForge compiler CLI (`agentforge compile <file.agent>`)
2. AgentForge runtime CLI (`agentforge run <file.agent>`)
3. Bedrock model routing (Claude 3.5 Sonnet, Llama 3 via Bedrock)
4. Built-in tools: `fs.glob`, `fs.read`, `fs.write`, `shell.run`, `human.checkpoint`, `retrieve()`
5. Web IDE: editor + compile + run + basic live dashboard
6. Pass-through billing: Stripe metered billing reconciled against Bedrock costs
7. User auth + pipeline storage (Postgres, hosted on AWS RDS)
8. Public landing page + sign-up flow

---

## Task Decomposition

### Milestone 1A: Compiler Core (Week 1–2)

**TASK-1-01: Implement `.agent` Lexer**
- Owner: agent
- Acceptance criteria: Tokenizes all grammar constructs defined in [language-spec.md](../05-specs/language-spec.md). 100% pass rate on the lexer test suite (min 50 test cases). Emits structured token stream with position info.
- Depends on: [language-spec.md](../05-specs/language-spec.md) finalized
- Effort: M

**TASK-1-02: Implement Recursive Descent Parser**
- Owner: agent
- Acceptance criteria: Produces a parse tree from a token stream for all grammar productions. Parse errors include line/column and expected token. 100% pass on parser test suite.
- Depends on: TASK-1-01
- Effort: L

**TASK-1-03: Build Task Graph AST**
- Owner: agent
- Acceptance criteria: Transforms parse tree into a TaskGraph with typed nodes (TaskNode, PipelineNode, LoopNode, BranchNode) and typed edges (data flow, control flow). Back-edge detection flags potential infinite loops as warnings.
- Depends on: TASK-1-02
- Effort: M

**TASK-1-04: Symbol Table and Scope Resolution**
- Owner: agent
- Acceptance criteria: All `$variable` references resolved to their declaration site. Loop variables scoped to loop body. Unresolved references emit compile error. Symbol table serializable to JSON for IDE tooling.
- Depends on: TASK-1-03
- Effort: M

**TASK-1-05: Type Checker**
- Owner: agent
- Acceptance criteria: Validates tool output types against downstream input types (see type definitions in language-spec.md). Emits typed error with expected/actual types and source location. 100% pass on type-checker test suite. Does not reject valid programs (zero false positives).
- Depends on: TASK-1-04
- Effort: L

**TASK-1-06: IR Emitter**
- Owner: agent
- Acceptance criteria: Emits IR as JSON array conforming to [runtime-spec.md](../05-specs/runtime-spec.md). All TaskGraph nodes produce valid IR instructions. IR is deterministic (same input → same output). Round-trip test: parse → IR → re-parse validates structural equivalence.
- Depends on: TASK-1-05
- Effort: M

**TASK-1-07: CLI `compile` command**
- Owner: agent
- Acceptance criteria: `agentforge compile <file.agent>` prints IR to stdout or `--output` file. Prints errors to stderr with exit code 1. Prints warnings to stderr with exit code 0. `--json` flag for structured output.
- Depends on: TASK-1-06
- Effort: S

### Milestone 1B: Runtime Dispatcher (Week 2–3)

**TASK-1-08: IR Walker / Dispatch Loop**
- Owner: agent
- Acceptance criteria: Walks IR instruction list in order. Handles `CALL`, `LLM_CALL`, `ASSIGN`, `BRANCH`, `LOOP_START`, `LOOP_END`, `CONTEXT_PUSH`, `RETRIEVE`. Emits structured execution events to a log stream. Handles panics with structured error and exit code 1.
- Depends on: TASK-1-06
- Effort: M

**TASK-1-09: Bedrock Model Adapter**
- Owner: agent
- Acceptance criteria: `LLM_CALL` with `model=claude-*` or `model=llama-*` dispatches to `bedrock:InvokeModelWithResponseStream`. Handles streaming responses. Tracks input/output tokens per call. Emits `ModelCallEvent` with latency and token counts.
- Depends on: TASK-1-08
- Effort: M

**TASK-1-10: Built-in Tool Implementations**
- Owner: agent
- Acceptance criteria: `fs.glob(pattern)` returns `FileList`. `fs.read(path)` returns `FileContent`. `fs.write(path, content)` returns `WriteResult`. `shell.run(cmd)` returns `ExecResult` with stdout, stderr, exit_code. `human.checkpoint(data)` pauses execution and emits a `CheckpointEvent`. `retrieve(path)` loads a markdown file from the OKF bundle. All tools have unit tests.
- Depends on: TASK-1-08
- Effort: M

**TASK-1-11: Context Budget Manager**
- Owner: agent
- Acceptance criteria: `CONTEXT_PUSH` tracks token count using tiktoken-compatible tokenizer. Emits warning at 80% of declared budget. Truncates oldest context entries (not most recent) when over budget. Budget accounting is per-LLM-call.
- Depends on: TASK-1-08
- Effort: S

**TASK-1-12: CLI `run` command**
- Owner: agent
- Acceptance criteria: `agentforge run <file.agent>` compiles and runs. `--dry-run` flag prints IR without executing. `--from-step <name>` resumes from a named step. Streams execution events to stdout as NDJSON. Exit code mirrors pipeline exit.
- Depends on: TASK-1-09, TASK-1-10, TASK-1-11
- Effort: S

### Milestone 1C: Web IDE + API (Week 3–5)

**TASK-1-13: REST API Server**
- Owner: agent
- Acceptance criteria: Implements endpoints: `POST /compile`, `POST /run`, `GET /runs/{id}`, `GET /runs/{id}/events` (SSE stream), `GET /pipelines`, `POST /pipelines`, `DELETE /pipelines/{id}`. Auth via JWT. Conforms to [api-spec.md](../05-specs/api-spec.md).
- Depends on: TASK-1-12
- Effort: L

**TASK-1-14: Web IDE — Editor**
- Owner: agent
- Acceptance criteria: Monaco editor with `.agent` syntax highlighting (keyword coloring, string literals, variable references). File browser panel. Save/load from pipeline storage API. No Intellisense in Phase 1 (Phase 2).
- Depends on: TASK-1-13
- Effort: M

**TASK-1-15: Web IDE — Compile + Run Panel**
- Owner: agent
- Acceptance criteria: "Compile" button calls `POST /compile`, displays errors/warnings inline in editor. "Run" button calls `POST /run`, opens live dashboard panel. `human.checkpoint` pauses execution and shows an approval dialog in the UI.
- Depends on: TASK-1-14
- Effort: M

**TASK-1-16: Web IDE — Live Dashboard**
- Owner: agent
- Acceptance criteria: Shows stage progress (current step, % complete for loops), token count vs. budget, cost-so-far (Bedrock rates), elapsed time. Real-time via SSE events. Step click shows raw LLM response + context sent. "Rerun from this step" button.
- Depends on: TASK-1-15
- Effort: L

### Milestone 1D: Auth + Billing (Week 5–7)

**TASK-1-17: User Auth (Clerk or Auth0)**
- Owner: agent
- Acceptance criteria: Sign up / sign in via email + Google OAuth. JWT tokens with user_id claim used by REST API. Free tier gating: 500 executions/month enforced at `POST /run`.
- Depends on: TASK-1-13
- Effort: S

**TASK-1-18: Stripe Metered Billing**
- Owner: founder (needs Stripe account + Bedrock billing access)
- Acceptance criteria: Stripe customer created on sign-up. Usage records posted to Stripe metered subscription after each run. Bedrock costs reconciled nightly via AWS Cost Explorer API. Pass-through invoice generated monthly. Overage gated at 110% of plan limit.
- Depends on: TASK-1-17
- Effort: L

**TASK-1-19: Postgres Schema + Migrations**
- Owner: agent
- Acceptance criteria: Schema per [data-model.md](../06-design/data-model.md). Alembic (or Flyway) migrations from zero. Tenant isolation: all queries scoped to `user_id`. Automated migration test in CI.
- Depends on: TASK-1-13
- Effort: M

### Milestone 1E: Launch Prep (Week 7–8)

**TASK-1-20: Landing Page**
- Owner: founder + contractor (copywriter/designer optional)
- Acceptance criteria: Above-the-fold: product name, one-sentence value prop, "Get started free" CTA. Below fold: code sample (the refactor pipeline from the press release), three benefit sections, pricing table, FAQ. Email capture. Mobile responsive.
- Depends on: nothing
- Effort: M

**TASK-1-21: CI/CD Pipeline**
- Owner: agent
- Acceptance criteria: GitHub Actions: lint + test on PR, deploy to staging on merge to `main`, deploy to production on tag. Compiler test suite, runtime test suite, and API integration tests all run in CI. < 5 min total CI time.
- Depends on: TASK-1-12, TASK-1-13
- Effort: S

**TASK-1-22: Onboarding Flow (3 steps)**
- Owner: agent
- Acceptance criteria: After sign-up: (1) paste or upload a `.agent` file, or choose from 3 starter templates. (2) Configure model (Bedrock Claude pre-selected, API key vault UI). (3) Click "Run" — see first execution live. Drop-off rate at each step measured.
- Depends on: TASK-1-15, TASK-1-17
- Effort: M

---

## Phase 1 Exit Criteria

See [go-no-go-criteria.md](../04-validation/go-no-go-criteria.md#phase-1-gate) for the full gate checklist. Summary:

- ≥ 10 users paying $29/month
- Time-to-first-run < 15 minutes (measured on 5 real users)
- Zero billing discrepancies between Stripe and Bedrock invoices
- Compiler false-positive rate < 5%
- Uptime ≥ 99.0% over 2-week observation window

---

## Related

- [Language Spec](../05-specs/language-spec.md) — required reading before TASK-1-01
- [Runtime Spec](../05-specs/runtime-spec.md) — required reading before TASK-1-08
- [API Spec](../05-specs/api-spec.md) — required reading before TASK-1-13
- [Data Model](../06-design/data-model.md) — required reading before TASK-1-19
- [Phase 2](phase-2-growth.md) — what comes next
