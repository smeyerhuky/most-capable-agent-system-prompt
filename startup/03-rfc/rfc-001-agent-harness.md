---
type: RFC
title: "RFC-001: AgentForge Agent Harness Architecture"
description: Technical architecture proposal for the AgentForge compiler pipeline, IR format, runtime dispatcher, model routing system, GPU serving layer, and multi-tenant cloud platform. This RFC is the authoritative technical design document for implementation.
tags: [rfc, architecture, compiler, runtime, ir, gpu, multi-tenant, bedrock, gemma4]
timestamp: 2026-06-28T00:00:00Z
---

# RFC-001: AgentForge Agent Harness Architecture

**Status:** Approved
**Author:** Founding team
**Reviewers:** (open for community review — see discussion)

---

## 1. Summary

This RFC proposes the technical architecture for AgentForge: a DSL compiler and cloud runtime for LLM-backed agent workflows. The system takes `.agent` source files through a classical compiler pipeline (lexer → parser → AST → type checker → IR) and dispatches the IR to a pluggable model router that targets AWS Bedrock (Claude, Llama 3), OpenAI, or a self-hosted Gemma 4 GPU cluster with MTP speculative decoding.

The compiler pipeline maps directly to classical compiler theory (Appel's *Modern Compiler Implementation*, Ch. 3–11), with agent-specific semantics: the symbol table tracks prompt variables, the type system encodes tool contracts, the IR is a dispatchable recipe rather than machine code, and the "VM" is an API orchestrator rather than a bytecode interpreter.

---

## 2. Motivation

Building reliable multi-step LLM agent workflows today is fragile, non-portable, and unauditable. Developers write Python/TypeScript with imperative API calls, hand-manage context windows, and have no compile-time validation. When a step fails, there is no structured understanding of why. When a model is swapped, orchestration code must be rewritten.

AgentForge addresses this with a language abstraction: declare intent, not implementation. The compiler enforces correctness; the runtime handles dispatch; the dashboard surfaces observability. The developer writes what they want, not how to make API calls.

---

## 3. Architecture Overview

```
.agent source
    │
    ▼
┌─────────┐    ┌─────────┐    ┌─────────────────┐    ┌──────────────┐    ┌──────┐
│  Lexer  │───►│ Parser  │───►│  AST + Symbol   │───►│ Type Checker │───►│  IR  │
│  (Ch.3) │    │  (Ch.4) │    │  Table (Ch.5-6) │    │    (Ch.7)    │    │(Ch.8)│
└─────────┘    └─────────┘    └─────────────────┘    └──────────────┘    └──┬───┘
                                                                             │
                                                                             ▼
                                                                    ┌─────────────────┐
                                                                    │  Runtime/VM     │
                                                                    │  Dispatcher     │
                                                                    │  (Ch.9/11)      │
                                                                    └────────┬────────┘
                                                                             │
                          ┌──────────────────────────────────────────────────┤
                          │                    │                              │
                    ┌─────▼──────┐    ┌────────▼───────┐    ┌───────────────▼──────┐
                    │  AWS       │    │  OpenAI        │    │  Private GPU         │
                    │  Bedrock   │    │  API           │    │  (vLLM + Gemma4 MTP) │
                    └────────────┘    └────────────────┘    └──────────────────────┘
```

---

## 4. Compiler Pipeline

### 4.1 Lexer

The lexer tokenizes `.agent` source into a flat token stream. All tokens carry source position (file, line, column) for error reporting.

Token categories:
- `KEYWORD`: `pipeline`, `step`, `model`, `context`, `prompt`, `tool`, `loop`, `for`, `in`, `branch`, `condition`, `output`, `on`
- `IDENT`: unquoted identifiers (step names, model names, variable names without `$`)
- `VAR`: `$identifier` — prompt variable references
- `STRING`: double-quoted string literals including glob patterns
- `TEMPLATE`: `{{ expression }}` — inline variable interpolation in prompt strings
- `COLON`, `ARROW` (`->`), `COMMA`, `LBRACKET`, `RBRACKET`, `LPAREN`, `RPAREN`
- `CALL`: `namespace.method(...)` — tool call expressions
- `NUMBER`: integer and float literals
- `COMMENT`: `#` to end of line (discarded)
- `NEWLINE`, `INDENT`, `DEDENT`: Python-style indentation tokens

### 4.2 Parser

Recursive descent parser. Grammar is described formally in [language-spec.md](../05-specs/language-spec.md). Key productions:

```
program        → pipeline_def*
pipeline_def   → 'pipeline' IDENT ':' NEWLINE INDENT pipeline_body DEDENT
pipeline_body  → (option_decl | step_def)*
step_def       → 'step' IDENT ':' NEWLINE INDENT step_body DEDENT
step_body      → (model_decl | context_decl | prompt_decl | tool_decl |
                  loop_decl | branch_decl | condition_decl | output_decl)*
```

Produces a `PipelineNode` containing `StepNode` children.

### 4.3 AST + Task Graph

The parse tree is rewritten into a `TaskGraph`:

```
TaskGraph {
  name: string
  options: { model_aliases: Map<string, ModelId>, context_budget: int }
  nodes: TaskNode[]
  edges: Edge[]
}

TaskNode {
  id: string                    // step name
  model?: ModelId
  context_refs: ContextRef[]    // ordered context declarations
  prompt?: PromptTemplate
  tool?: ToolCall
  loop?: LoopSpec               // { variable: Var, iterable: Var }
  output?: VarDecl
}

Edge {
  from: string
  to: string
  kind: 'data' | 'control' | 'conditional'
  label?: string                // condition label (e.g., "approved", "exit_code == 0")
  data?: VarRef[]               // variables carried on this edge
}
```

Back-edges (cycles) are detected via DFS and flagged as warnings (not errors — loops are intentional).

### 4.4 Symbol Table

The symbol table resolves all `$variable` references. Scoping rules:

- **Pipeline scope:** Variables declared in a step's `output:` are visible to all subsequent steps
- **Loop scope:** The loop iteration variable (e.g., `$item`) is scoped to the loop body
- **Shadow rule:** Loop variables shadow pipeline-scope variables of the same name (warning emitted)
- **Forward reference:** A variable used before its declaring step is a compile error (unresolvable at the resolving step's execution)

Symbol table is serialized to JSON and consumed by the LSP for IDE Intellisense.

### 4.5 Type System

The type system validates tool contracts between steps. Core types:

| Type | Description |
|---|---|
| `FileList` | List of file paths (from `fs.glob`) |
| `FileContent` | String content of a file (from `fs.read`) |
| `WriteResult` | Success/error from `fs.write` |
| `ExecResult` | `{ stdout: string, stderr: string, exit_code: int }` |
| `ApprovalResult` | `{ status: "approved" \| "rejected", note?: string }` |
| `IssueList` | LLM-generated structured output (JSON validated against schema) |
| `Issue` | Single item from an `IssueList` |
| `MarkdownContent` | String (untyped LLM output) |
| `ModelOutput<T>` | Generic LLM output, parameterized by declared output type |

Type inference: if a step's `prompt:` specifies `output: $var` and declares `output: json_list`, the variable is typed as `IssueList`. If no type annotation, defaults to `MarkdownContent`.

Type errors are compile-time errors with exact expected/actual types and the source location of both the declaration and the use.

### 4.6 IR Format

The IR is a JSON array of instruction objects. Full instruction set in [runtime-spec.md](../05-specs/runtime-spec.md). Key instructions:

```json
{ "op": "CALL", "target": "fs.glob", "args": ["./src/**/*.ts"], "result": "tmp0" }
{ "op": "ASSIGN", "var": "$source_files", "src": "tmp0" }
{ "op": "RETRIEVE", "path": "wiki/concepts/symbol-table.md", "result": "tmp1" }
{ "op": "CONTEXT_PUSH", "refs": ["$source_files", "tmp1"], "budget": 32000 }
{ "op": "LLM_CALL", "model": "claude-3-5-sonnet-20241022", "prompt": "...", "result": "tmp2" }
{ "op": "ASSIGN", "var": "$dead_items", "src": "tmp2" }
{ "op": "BRANCH", "src": "tmp3.status", "cases": [{"eq": "approved", "target": 10}, {"eq": "rejected", "target": 24}] }
{ "op": "LOOP_START", "iterable": "$dead_items", "var": "$item", "end_label": "LOOP_1_END" }
{ "op": "LOOP_END", "label": "LOOP_1_END", "start_index": 8 }
{ "op": "EMIT", "message": "Refactor complete. {{ len($dead_items) }} items removed." }
```

---

## 5. Runtime Dispatcher

The runtime is a sequential IR interpreter with async dispatch for LLM calls and tool calls.

### 5.1 Execution Model

- IR is walked instruction by instruction
- `LLM_CALL` and `CALL` dispatch asynchronously and `await` completion
- `BRANCH` updates the instruction pointer
- `LOOP_START` / `LOOP_END` maintain an iteration counter and loop stack
- `CONTEXT_PUSH` accumulates a context buffer, reset at each `LLM_CALL`
- `human.checkpoint` emits a `CheckpointEvent` and suspends execution until an `ApproveEvent` or `RejectEvent` is received from the UI layer

Execution state (current IP, variable bindings, context buffer) is serialized to Redis after each instruction for recovery. A run can be resumed from any instruction.

### 5.2 Model Router

The model router resolves `model:` declarations to backend adapters:

```
ModelRouter {
  aliases: Map<string, ModelId>    // from pipeline options
  adapters: Map<ModelId, Adapter>  // registered adapters
  fallback: Adapter                // used if primary fails
}

Adapter interface:
  async call(prompt: string, context: string[], options: CallOptions): ModelResponse
  tokenCount(text: string): int
  name(): string
```

Registered adapters: `BedrockAdapter`, `OpenAIAdapter`, `GemmaVLLMAdapter`.

### 5.3 Tool Registry

Built-in tools are registered at startup. Tool calls are resolved at dispatch time:

```
ToolRegistry {
  tools: Map<string, ToolFn>
}

ToolFn: (args: any[], ctx: ExecutionContext) => Promise<TypedValue>
```

MCP tools are registered dynamically at compile time (schema fetch) and dispatch time (MCP protocol call).

### 5.4 Event Stream

Every IR instruction emits an `ExecutionEvent` to an event stream (Redis Pub/Sub → SSE → client). Event types:

- `StepStarted`, `StepCompleted`, `StepFailed`
- `LLMCallStarted`, `LLMCallCompleted` (includes tokens in/out, latency, model)
- `ToolCallStarted`, `ToolCallCompleted` (includes tool name, result summary)
- `CheckpointPaused`, `CheckpointApproved`, `CheckpointRejected`
- `ContextPushed` (includes token count, budget remaining)
- `BranchTaken` (includes condition label)
- `LoopIteration` (includes index, total)
- `RunCompleted`, `RunFailed`

---

## 6. GPU Serving Layer

See [gpu-serving-design.md](../06-design/gpu-serving-design.md) for full details.

Summary: vLLM with speculative decoding serving `google/gemma-4-26B-A4B-it` on AWS `g5.xlarge` spot instances. The drafter model (`google/gemma-4-26B-A4B-it-assistant`) uses the target model's KV cache activations — resulting in higher draft acceptance rates than generic speculative decoding. Target: 50–70 tokens/sec on A10G with MTP heuristic schedule enabled.

The vLLM endpoint exposes an OpenAI-compatible API. The `GemmaVLLMAdapter` is a thin wrapper that sets `base_url` to the vLLM host and delegates to the OpenAI client SDK.

---

## 7. Multi-Tenant Isolation

- Every database query is scoped to `user_id` or `team_id` (enforced at ORM layer)
- API keys stored encrypted (AES-256-GCM, key per tenant in AWS KMS)
- Run executions are isolated: each run gets a fresh execution context with no shared mutable state
- File tool calls (`fs.*`) are sandboxed: writes are staged to a per-run temp directory and committed atomically, preventing cross-user file access
- GPU tier: requests are queued per-user with a per-user concurrency limit of 3

---

## 8. Deployment Architecture

See [system-architecture.md](../06-design/system-architecture.md) for full diagrams.

Key components:
- **Control plane:** ECS Fargate (API server, compiler service, event bus relay)
- **Database:** RDS Postgres (multi-AZ, Phase 3), ElastiCache Redis (event bus + execution state)
- **GPU fleet:** `g5.xlarge` spot instances managed by an Auto Scaling Group, fronted by an internal ALB
- **Object storage:** S3 for pipeline sources, IR files, run artifacts
- **CDN:** CloudFront for web IDE static assets
- **Monitoring:** CloudWatch + OpenTelemetry → Grafana

---

## 9. Open Questions

1. **Concurrency model for loops:** Should loop iterations execute in parallel (fan-out) or sequentially? Sequential is simpler and avoids context window conflicts; parallel is faster but requires per-iteration context isolation. **Proposed:** Sequential in Phase 1, fan-out as an opt-in annotation (`loop parallel:`) in Phase 2.

2. **Grammar extensibility:** Should the language support user-defined tool types? **Proposed:** Phase 2 introduces `tool_def` blocks that let users declare typed tool wrappers around MCP servers, with the type checker validating them as first-class tools.

3. **IR versioning:** The IR format must be versioned for backward compatibility as the language evolves. **Proposed:** IR includes a `version: "1.0"` field. Runtime refuses to execute IR with a major version it doesn't understand.

4. **Prompt template security:** `{{ expr }}` in prompt strings could be used for prompt injection if `expr` contains user-supplied data. **Proposed:** Variables interpolated via `{{ }}` are sanitized (no backtick nesting, no system prompt injection markers) at IR emit time.

---

## 10. References

- [Language Spec](../05-specs/language-spec.md) — EBNF grammar and type definitions
- [Runtime Spec](../05-specs/runtime-spec.md) — complete IR instruction set
- [Compiler Design](../06-design/compiler-design.md) — implementation-level design
- [GPU Serving Design](../06-design/gpu-serving-design.md) — vLLM + MTP architecture
- [PRD Phase 1](../02-prd/phase-1-mvp.md) — task decomposition implementing this RFC
