---
type: PRD Phase
title: "Phase 2 — Growth"
description: Growth phase requirements and agentic PDLC task decomposition for AgentForge. Covers VS Code extension, MCP protocol support, GitHub Actions integration, and the pipeline template marketplace. Target: 500 GitHub stars, 1,000 VS Code installs, $2,900 MRR.
tags: [prd, growth, phase-2, tasks, vscode, mcp, github-actions, marketplace]
timestamp: 2026-06-28T00:00:00Z
---

# Phase 2 — Growth

**Duration:** Months 3–4 (8 weeks)
**Gate:** [Phase 2 Go/No-Go](../04-validation/go-no-go-criteria.md#phase-2-gate)
**Goal:** Developer ecosystem adoption. 500 GitHub stars, 1,000 VS Code extension installs, 100 monthly active pipelines, and $2,900 MRR (100 Builder subscribers).

---

## What Ships in Phase 2

1. VS Code extension with Intellisense, inline errors, and one-click run
2. MCP protocol support (AgentForge as MCP server + MCP tools as first-class citizens)
3. GitHub Actions integration (`agentforge run` in CI)
4. Pipeline template marketplace (10 built-in templates, community submission)
5. HN Show HN launch + GitHub public repo open-source of the compiler
6. Improved onboarding: AI-assisted `.agent` file generator
7. OpenAI adapter (adds GPT-4o as a routing target)
8. WebSocket live dashboard (upgrade from SSE for lower latency)

---

## Task Decomposition

### Milestone 2A: VS Code Extension (Week 9–11)

**TASK-2-01: VS Code Extension Scaffold**
- Owner: agent
- Acceptance criteria: Extension published to VS Code Marketplace under publisher `agentforge`. Activates on `.agent` file open. No Intellisense yet (added in TASK-2-03). Extension installs cleanly from VSIX and from marketplace.
- Depends on: TASK-1-07
- Effort: S

**TASK-2-02: Syntax Highlighting (TextMate Grammar)**
- Owner: agent
- Acceptance criteria: `.agent` grammar file covers: keywords (`pipeline`, `step`, `model`, `context`, `prompt`, `tool`, `loop`, `branch`, `condition`, `output`), variables (`$identifier`), string literals, comments (`#`), template expressions (`{{ expr }}`), builtin tool calls. All tokens colored correctly in both light and dark VS Code themes.
- Depends on: TASK-2-01
- Effort: S

**TASK-2-03: Language Server (LSP)**
- Owner: agent
- Acceptance criteria: LSP server wraps the compiler's symbol table. Provides: hover documentation for builtins and keywords, go-to-definition for `$variables`, inline error/warning squiggles using compile output, auto-complete for keywords and variable names. Latency < 200ms for all LSP responses.
- Depends on: TASK-1-04, TASK-2-01
- Effort: L

**TASK-2-04: One-Click Run from VS Code**
- Owner: agent
- Acceptance criteria: "Run Pipeline" button in VS Code command palette and title bar. Opens a VS Code webview panel showing the live dashboard (mirrors web IDE dashboard). `human.checkpoint` pauses show an inline dialog in VS Code. Run status shown in VS Code status bar.
- Depends on: TASK-2-03, TASK-1-13
- Effort: M

### Milestone 2B: MCP Protocol (Week 10–12)

**TASK-2-05: MCP Server — AgentForge Exposes Tools**
- Owner: agent
- Acceptance criteria: AgentForge runs as an MCP server. Exposes: `agentforge.compile(source: string)`, `agentforge.run(pipeline_id: string)`, `agentforge.list_pipelines()`, `agentforge.get_run_status(run_id: string)`. MCP clients (Claude Desktop, Cursor) can call these tools. Schema follows MCP v1.0 spec.
- Depends on: TASK-1-13
- Effort: M

**TASK-2-06: MCP Tool Consumer — Use MCP Tools in `.agent` Files**
- Owner: agent
- Acceptance criteria: `.agent` DSL supports `tool: mcp:<server>/<tool>` syntax. Compiler resolves MCP tool schema at compile time (fetches schema from server). Type checker validates MCP tool input/output against surrounding step types. Runtime dispatches to MCP server at execution time. Example: GitHub MCP tools usable inline in pipelines.
- Depends on: TASK-1-10, TASK-2-05
- Effort: L

**TASK-2-07: MCP Tool Auth Vault**
- Owner: agent
- Acceptance criteria: Web IDE settings page: user adds MCP server URLs and API keys. Keys stored encrypted in Postgres (AES-256). Runtime fetches keys at dispatch time. Keys never exposed in logs or dashboard. Key rotation UI.
- Depends on: TASK-2-06, TASK-1-17
- Effort: M

### Milestone 2C: GitHub Actions (Week 11–12)

**TASK-2-08: `agentforge/run` GitHub Action**
- Owner: agent
- Acceptance criteria: Published to GitHub Marketplace as `agentforge/run@v1`. Inputs: `pipeline` (path to `.agent` file), `api-key` (secret), `model` (optional override). On failure, outputs compiler errors and runtime errors as GitHub step summary. On success, outputs run_id and summary. Works in pull_request, push, and workflow_dispatch triggers.
- Depends on: TASK-1-12
- Effort: M

**TASK-2-09: CI/CD Pipeline Template**
- Owner: agent
- Acceptance criteria: Built-in `.agent` template "CI Refactor Validator" that: runs on PR, analyzes diff for regressions, suggests fixes, posts a summary comment to the PR via GitHub MCP. Template is published to the marketplace in TASK-2-10.
- Depends on: TASK-2-06, TASK-2-08
- Effort: M

### Milestone 2D: Template Marketplace (Week 12–13)

**TASK-2-10: Marketplace Backend**
- Owner: agent
- Acceptance criteria: `GET /marketplace/templates` returns paginated list with name, description, author, download count, tags. `GET /marketplace/templates/{id}/download` returns `.agent` source. `POST /marketplace/templates` (authenticated) submits a template for review. Founder review queue UI for approving submissions.
- Depends on: TASK-1-13, TASK-1-19
- Effort: M

**TASK-2-11: 10 Built-In Templates**
- Owner: agent + founder
- Acceptance criteria: 10 templates covering: code refactor, dead code finder, test generator, PR review assistant, docs generator, API migration helper, dependency auditor, CI validator, security scanner, performance profiler. Each template has a README with example output. All templates pass compile + type check.
- Depends on: TASK-2-10, TASK-1-10
- Effort: L

**TASK-2-12: Marketplace Discovery UI**
- Owner: agent
- Acceptance criteria: Marketplace page in web IDE: search, filter by tags, sort by downloads/recency. "Use template" one-click imports to editor. Template previews show first 30 lines of source. Author attribution and link to GitHub profile.
- Depends on: TASK-2-10, TASK-1-14
- Effort: M

### Milestone 2E: OpenAI Adapter + AI Generator (Week 13–14)

**TASK-2-13: OpenAI Model Adapter**
- Owner: agent
- Acceptance criteria: `LLM_CALL` with `model=gpt-4o` or `model=gpt-4o-mini` dispatches to OpenAI Chat Completions API. Streaming responses. Token tracking. API key configured in user settings (not pass-through — user's own OpenAI key). Adapter is pluggable (same interface as Bedrock adapter).
- Depends on: TASK-1-09
- Effort: M

**TASK-2-14: AI-Assisted `.agent` Generator**
- Owner: agent
- Acceptance criteria: "Generate Pipeline" button in web IDE. User describes what they want in natural language. Claude (via Bedrock, using AgentForge's own API key) generates a `.agent` file. Generator prompt is grounded with the full language spec and 3 example pipelines. Output is automatically compiled and errors fed back to the model for one self-correction pass. Generated pipelines pass the type checker > 80% of the time.
- Depends on: TASK-1-09, TASK-1-15
- Effort: M

### Milestone 2F: Launch (Week 15–16)

**TASK-2-15: Open-Source Compiler**
- Owner: founder (GitHub repo creation + license decision)
- Acceptance criteria: Compiler (lexer, parser, AST, type checker, IR emitter) published under MIT license on GitHub. README with 5-minute quickstart. Contributing guide. Issue templates. GitHub Discussions enabled. 3 "good first issue" issues filed on launch day.
- Depends on: TASK-1-07
- Effort: S

**TASK-2-16: HN Show HN Post**
- Owner: founder
- Acceptance criteria: Show HN post per template in [customer-acquisition.md](../07-business-plan/customer-acquisition.md). Posted on a Tuesday or Wednesday at 9am EST. Founder available for 4 hours of comment responses post-launch.
- Depends on: TASK-2-15, TASK-2-11
- Effort: S

---

## Phase 2 Exit Criteria

- ≥ 500 GitHub stars on the open-source compiler
- ≥ 1,000 VS Code extension installs
- ≥ 100 monthly active pipelines (pipeline with ≥ 1 run in the last 30 days)
- ≥ $2,900 MRR
- MCP integration tested with ≥ 3 third-party MCP servers
- ≥ 10 pipeline templates in marketplace

---

## Related

- [Phase 1 MVP](phase-1-mvp.md) — prerequisite
- [Phase 3 Scale](phase-3-scale.md) — what comes next
- [Customer Acquisition](../07-business-plan/customer-acquisition.md) — HN post template and launch checklist
- [Go-to-Market](../07-business-plan/go-to-market.md) — channel strategy for Phase 2 launch
