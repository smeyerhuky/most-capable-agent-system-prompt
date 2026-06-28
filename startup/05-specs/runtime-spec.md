---
type: Spec
title: "AgentForge Runtime Specification v0.1.0"
description: Formal specification for the AgentForge IR instruction set, execution model, tool dispatch contract, context budget accounting, and execution event stream. Version 0.1.0.
tags: [spec, runtime, ir, execution-model, dispatch, events, context-budget]
timestamp: 2026-06-28T00:00:00Z
---

# AgentForge Runtime Specification v0.1.0

---

## 1. IR Format

The IR is a JSON document with the following structure:

```json
{
  "version": "1.0",
  "pipeline": "PipelineName",
  "options": {
    "context_budget": 32000,
    "model_aliases": {
      "default": "claude-3-5-sonnet-20241022",
      "fast": "gemma4-26b-private"
    }
  },
  "instructions": [
    { "index": 0, "op": "...", ... },
    { "index": 1, "op": "...", ... }
  ]
}
```

`version` is a semantic version string. The runtime refuses to execute IR with a major version it does not support.

---

## 2. Instruction Set

### CALL

Dispatches a built-in tool call.

```json
{ "op": "CALL", "index": 0, "target": "fs.glob", "args": ["./src/**/*.ts"], "result": "tmp0" }
```

- `target`: `namespace.method` string
- `args`: ordered list of string, number, or variable reference (prefixed `$`)
- `result`: temporary register name for the return value

### LLM_CALL

Dispatches an LLM inference request.

```json
{
  "op": "LLM_CALL",
  "index": 4,
  "model": "claude-3-5-sonnet-20241022",
  "prompt": "Identify unused exports...",
  "context_register": "ctx0",
  "result": "tmp2",
  "output_type": "json_list"
}
```

- `model`: resolved model ID (after alias lookup)
- `context_register`: the context buffer register populated by `CONTEXT_PUSH` instructions before this call
- `output_type`: `null` (MarkdownContent) | `"json"` | `"json_list"`

### ASSIGN

Binds a named variable to a temporary register or literal.

```json
{ "op": "ASSIGN", "index": 1, "var": "$source_files", "src": "tmp0" }
```

### RETRIEVE

Loads a markdown file from the knowledge bundle.

```json
{ "op": "RETRIEVE", "index": 2, "path": "wiki/concepts/symbol-table.md", "result": "tmp1" }
```

### CONTEXT_PUSH

Appends items to the current context buffer for the next `LLM_CALL`.

```json
{
  "op": "CONTEXT_PUSH",
  "index": 3,
  "items": ["$source_files", "tmp1"],
  "budget": 32000,
  "register": "ctx0"
}
```

- `items`: list of variable or temporary register names to include in context
- `budget`: token budget for this context window
- `register`: context buffer register (reset after each `LLM_CALL` that consumes it)

The runtime tokenizes each item, tracks cumulative token count, and truncates if the budget would be exceeded. Truncation strategy: drop the oldest items first (not the most recent). Emit `W003` warning when truncation occurs.

### BRANCH

Conditional jump based on a value.

```json
{
  "op": "BRANCH",
  "index": 7,
  "src": "tmp3.status",
  "cases": [
    { "eq": "approved", "target": 10 },
    { "eq": "rejected", "target": 24 }
  ],
  "default_target": 24
}
```

- `src`: a variable reference or `register.field` path
- `cases`: list of `{ eq | ne | lt | gt | lte | gte, value, target }` objects
- `default_target`: instruction index to jump to if no case matches

### LOOP_START

Begins a loop over a list variable.

```json
{
  "op": "LOOP_START",
  "index": 8,
  "iterable": "$dead_items",
  "var": "$item",
  "end_label": "LOOP_1_END",
  "end_target": 14
}
```

The runtime pushes a loop frame onto the loop stack: `{ iterable, var, index: 0, end_target }`. On each iteration, `$item` is bound to `iterable[index]` and `index` is incremented.

### LOOP_END

Marks the end of a loop body.

```json
{ "op": "LOOP_END", "index": 13, "label": "LOOP_1_END", "start_target": 8 }
```

The runtime checks the top loop frame. If `index < len(iterable)`, jumps to `start_target` (next iteration). Otherwise pops the loop frame and continues to `index + 1`.

### EMIT

Emits a final output message.

```json
{ "op": "EMIT", "index": 22, "message": "Refactor complete. {{ len($dead_items) }} items removed." }
```

Template expressions in `message` are evaluated at runtime.

### CHECKPOINT

Pauses execution for human approval.

```json
{ "op": "CHECKPOINT", "index": 6, "data_ref": "$dead_items", "result": "tmp3" }
```

Execution suspends. The runtime emits a `CheckpointPaused` event with the serialized `data_ref` value. Resumes when an `ApproveEvent` or `RejectEvent` arrives. `result` is bound to an `ApprovalResult`.

---

## 3. Execution State

The runtime maintains the following state per run:

```
ExecutionState {
  run_id: uuid
  pipeline: string
  ip: int                           // instruction pointer
  variables: Map<string, TypedValue>  // named variables ($name)
  registers: Map<string, TypedValue>  // temporary registers (tmp0, tmp1, ...)
  context_buffers: Map<string, ContextBuffer>  // ctx0, etc.
  loop_stack: LoopFrame[]
  status: 'running' | 'paused' | 'completed' | 'failed'
  created_at: timestamp
  updated_at: timestamp
}
```

Execution state is persisted to Redis after every instruction completion. Runs can be resumed from any instruction via `POST /runs/{id}/resume`.

---

## 4. Tool Dispatch Contract

Built-in tools must satisfy the following contract:

1. **Deterministic typing:** The return type of a tool call is determined entirely from its argument types at compile time.
2. **Error handling:** Tool failures must return a structured error value rather than throwing. The runtime wraps tool calls in try/catch. On error, the run transitions to `failed` status and emits a `ToolCallFailed` event with the error.
3. **Timeout:** All tool calls have a maximum execution time of 300 seconds. Exceeded calls are cancelled and treated as errors.
4. **Sandboxing:** `fs.*` tool calls are restricted to the working directory (no `../` traversal). `shell.run()` executes in an isolated subprocess with no access to the runtime's environment variables.
5. **Idempotency:** `fs.write()` is idempotent if the content is unchanged. `shell.run()` is not guaranteed idempotent — callers are responsible for idempotent commands in retry scenarios.

---

## 5. Context Budget Accounting

Token counting uses the cl100k_base tokenizer (compatible with Claude and GPT-4 token counts). Budget accounting:

1. At each `CONTEXT_PUSH`, each item in `items` is tokenized independently.
2. The running total is compared against `budget`.
3. If adding an item would exceed the budget, it is dropped and `W003` is emitted to the event stream.
4. Truncation order: items are dropped in insertion order, oldest first (items declared first in the `context:` block are dropped first if the budget is exceeded).
5. The `LLM_CALL` always includes the `prompt` text in addition to the context buffer. Prompt token count is included in the budget accounting.

---

## 6. Execution Event Stream

Events are emitted to the event stream as NDJSON. Each event has:

```json
{ "event": "EventType", "run_id": "uuid", "timestamp": "ISO8601", "data": { ... } }
```

### Event Types

| Event | Data Fields | When |
|---|---|---|
| `RunStarted` | `pipeline, options` | Before first instruction |
| `StepStarted` | `step_name, ip` | When IP enters a step's first instruction |
| `StepCompleted` | `step_name, elapsed_ms` | When all step instructions complete |
| `StepFailed` | `step_name, error` | On unhandled error in a step |
| `LLMCallStarted` | `model, prompt_tokens, context_tokens` | Before model API call |
| `LLMCallCompleted` | `model, output_tokens, elapsed_ms, cost_usd` | After model response received |
| `LLMCallFailed` | `model, error, elapsed_ms` | On model API error |
| `ToolCallStarted` | `tool, args_summary` | Before tool execution |
| `ToolCallCompleted` | `tool, result_type, elapsed_ms` | After tool returns |
| `ToolCallFailed` | `tool, error, elapsed_ms` | On tool error |
| `CheckpointPaused` | `data_preview` | When `CHECKPOINT` suspends execution |
| `CheckpointApproved` | `note?` | When user approves |
| `CheckpointRejected` | `note?` | When user rejects |
| `ContextPushed` | `tokens_added, total_tokens, budget, truncated` | After `CONTEXT_PUSH` |
| `BranchTaken` | `condition, target_step` | After `BRANCH` resolves |
| `LoopIteration` | `var, index, total` | At each `LOOP_START` iteration |
| `RunCompleted` | `elapsed_ms, total_cost_usd, total_tokens` | After terminal step |
| `RunFailed` | `error, ip, elapsed_ms` | On unrecoverable runtime error |
| `Warning` | `code, message, ip` | On W001–W004 warnings |

---

## 7. Resumption Protocol

A paused run (status `paused` due to `CHECKPOINT`) can be resumed via:

```
POST /runs/{id}/resume
Body: { "action": "approved" | "rejected", "note": "optional user note" }
```

The runtime receives the action, binds the `ApprovalResult` to the checkpoint's result register, and continues execution from `ip + 1`.

---

## Related

- [Language Spec](language-spec.md) — the language compiled to this IR
- [API Spec](api-spec.md) — the API that submits runs and streams events
- [Runtime Design](../06-design/runtime-design.md) — implementation of this spec
