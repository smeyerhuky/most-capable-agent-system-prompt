---
type: Spec
title: "AgentForge Language Specification v0.1.0"
description: Formal specification for the AgentForge .agent DSL. Covers EBNF grammar, type system, built-in standard library, error codes, and runtime contract. Version 0.1.0.
tags: [spec, language, dsl, grammar, ebnf, type-system, stdlib, agent-files]
timestamp: 2026-06-28T00:00:00Z
---

# AgentForge Language Specification v0.1.0

---

## 1. Introduction

The `.agent` file format is a declarative domain-specific language for defining multi-step LLM agent workflows. Programs written in `.agent` are compiled to an Intermediate Representation (IR) and executed by the AgentForge runtime.

A `.agent` program consists of one or more `pipeline` declarations. Each pipeline contains `step` declarations that define the agent's tasks, the tools they use, the models they call, and the data they produce.

---

## 2. Lexical Structure

### 2.1 Character Set

Source files are UTF-8 encoded.

### 2.2 Comments

Single-line comments begin with `#` and extend to the end of the line. Comments are discarded during tokenization.

### 2.3 Whitespace and Indentation

AgentForge uses Python-style significant indentation. `INDENT` and `DEDENT` tokens are emitted when indentation increases or decreases. Tabs are forbidden; indentation must use spaces (2 or 4, consistent within a file).

### 2.4 Identifiers

```
IDENT ::= [a-zA-Z_][a-zA-Z0-9_]*
```

Identifiers are case-sensitive. Keywords are reserved and cannot be used as identifiers.

### 2.5 Variables

```
VAR ::= '$' IDENT
```

Variable names are prefixed with `$`. Variables hold typed values produced by step outputs and tool calls.

### 2.6 String Literals

```
STRING ::= '"' (char | ESCAPE | TEMPLATE_EXPR)* '"'
ESCAPE ::= '\\' ['"\\nrt]
TEMPLATE_EXPR ::= '{{' EXPR '}}'
```

Template expressions (`{{ expr }}`) are evaluated at runtime. Valid expressions in templates: variable references (`$var`), property access (`$var.field`), and built-in functions (`len($var)`, `str($var)`).

### 2.7 Keywords

```
pipeline  step  model  context  budget  prompt  tool  output
loop      for   in     branch   on      condition  retrieve
```

### 2.8 Operators and Punctuation

```
:  ->  ,  [  ]  (  )  ==  !=  <  >  <=  >=
```

---

## 3. Grammar (EBNF)

```ebnf
program         ::= pipeline_def*

pipeline_def    ::= 'pipeline' IDENT ':' NEWLINE
                    INDENT pipeline_body DEDENT

pipeline_body   ::= pipeline_option* step_def*

pipeline_option ::= model_alias
                  | context_budget

model_alias     ::= 'model' IDENT ':' model_id
model_id        ::= STRING  (* e.g. "claude-3-5-sonnet", "gemma4-26b-private" *)

context_budget  ::= 'context' 'budget' ':' NUMBER 'tokens'

step_def        ::= 'step' IDENT ':' NEWLINE
                    INDENT step_body DEDENT

step_body       ::= step_clause*

step_clause     ::= model_ref
                  | context_decl
                  | prompt_decl
                  | tool_call
                  | loop_decl
                  | branch_decl
                  | condition_decl
                  | output_decl

model_ref       ::= 'model' ':' IDENT

context_decl    ::= 'context' ':' NEWLINE
                    INDENT context_item+ DEDENT
                  | 'context' ':' '[' context_item (',' context_item)* ']'

context_item    ::= VAR
                  | tool_call
                  | 'retrieve' '(' STRING ')'

prompt_decl     ::= 'prompt' ':' STRING

tool_call       ::= 'tool' ':' qualified_call

qualified_call  ::= IDENT '.' IDENT '(' arg_list? ')'

arg_list        ::= arg (',' arg)*

arg             ::= STRING | NUMBER | VAR | qualified_call

output_decl     ::= 'output' ':' VAR

loop_decl       ::= 'loop' ':' 'for' VAR 'in' VAR NEWLINE
                    INDENT step_body DEDENT

branch_decl     ::= 'branch' ':' NEWLINE
                    INDENT branch_arm+ DEDENT

branch_arm      ::= 'on' ':' condition_expr '->' IDENT

condition_expr  ::= STRING  (* simple label: "approved", "rejected" *)
                  | VAR '.' IDENT comparison_op (NUMBER | STRING)
                  (* e.g., $result.exit_code == 0 *)

comparison_op   ::= '==' | '!=' | '<' | '>' | '<=' | '>='

condition_decl  ::= 'condition' ':' NEWLINE
                    INDENT branch_arm+ DEDENT
```

---

## 4. Type System

### 4.1 Primitive Types

| Type | Description | Produced By |
|---|---|---|
| `FileList` | Ordered list of file paths (strings) | `fs.glob()` |
| `FileContent` | UTF-8 string content of a single file | `fs.read()` |
| `WriteResult` | `{ path: string, bytes_written: int }` | `fs.write()` |
| `ExecResult` | `{ stdout: string, stderr: string, exit_code: int }` | `shell.run()` |
| `ApprovalResult` | `{ status: "approved" \| "rejected", note?: string }` | `human.checkpoint()` |
| `MarkdownContent` | Unstructured LLM text output | LLM calls without output annotation |
| `JsonObject` | Parsed JSON object | LLM calls with `output: json` annotation |
| `JsonList` | Parsed JSON array | LLM calls with `output: json_list` annotation |
| `String` | Plain string | String literals, template expressions |
| `Int` | Integer | Numeric literals |

### 4.2 Generic Types

| Type | Description |
|---|---|
| `List<T>` | Ordered list of `T`. `FileList` = `List<String>`. |
| `Item<T>` | Single element drawn from `List<T>` during loop iteration |
| `ModelOutput<T>` | LLM output typed as `T`. The type checker validates that downstream uses are compatible with `T`. |

### 4.3 Type Inference Rules

1. `fs.glob(pattern: String) → FileList`
2. `fs.read(path: String | Item<FileList>) → FileContent`
3. `fs.write(path: String | Item<FileList>, content: String) → WriteResult`
4. `shell.run(command: String) → ExecResult`
5. `human.checkpoint(data: Any) → ApprovalResult`
6. `retrieve(path: String) → MarkdownContent`
7. LLM call without output annotation → `MarkdownContent`
8. LLM call with prompt suffix `"... Return JSON."` and `output: json_list` → `JsonList`
9. Loop variable `$item` over `JsonList` → `JsonObject`
10. Property access `$result.exit_code` on `ExecResult` → `Int`
11. Property access `$result.status` on `ApprovalResult` → `String`

### 4.4 Type Errors

The type checker emits an error (not a warning) for:
- Passing a `FileList` where `FileContent` is expected
- Accessing `.exit_code` on a non-`ExecResult` type
- Referencing an unresolved variable
- Branching on a field that does not exist in the source type
- Using a loop variable outside its loop body

---

## 5. Standard Library

### 5.1 Filesystem Tools

**`fs.glob(pattern: String) → FileList`**
Returns all file paths matching the glob pattern relative to the working directory. Glob syntax follows the `fnmatch` standard (e.g., `./src/**/*.ts`).

**`fs.read(path: String) → FileContent`**
Returns the UTF-8 contents of the file at `path`. Errors if file does not exist.

**`fs.write(path: String, content: String) → WriteResult`**
Writes `content` to `path`. Creates parent directories if needed. Returns write metadata. In dry-run mode, writes are staged but not committed.

### 5.2 Shell Tools

**`shell.run(command: String) → ExecResult`**
Executes `command` in a subprocess using `/bin/sh -c`. Captures stdout and stderr. Returns `ExecResult` with exit code. Timeout: 300 seconds. The command runs in an isolated subprocess — not the same shell session between calls.

### 5.3 Human Tools

**`human.checkpoint(data: Any) → ApprovalResult`**
Pauses execution and surfaces `data` to the user via the dashboard or VS Code panel. Execution resumes when the user clicks "Approve" or "Reject". In non-interactive mode (CI), defaults to `rejected` after a configurable timeout (default: 300 seconds).

### 5.4 Retrieval

**`retrieve(path: String) → MarkdownContent`**
Loads a markdown file from the OKF knowledge bundle bundled with the pipeline. Useful for grounding LLM prompts with documentation, specs, or reference material. The path is relative to the pipeline file.

### 5.5 MCP Tools (Phase 2+)

**`mcp:<server>/<tool>(<args>) → <declared_return_type>`**
Dispatches to an MCP server registered in the user's tool vault. The tool schema is fetched from the MCP server at compile time for type checking. At runtime, the call is dispatched via the MCP protocol.

---

## 6. Error Codes

| Code | Category | Description |
|---|---|---|
| E001 | Lexer | Unexpected character |
| E002 | Lexer | Unterminated string literal |
| E003 | Lexer | Invalid indentation (mixed tabs/spaces) |
| E010 | Parser | Expected keyword, got `X` |
| E011 | Parser | Expected `IDENT`, got `X` |
| E012 | Parser | Unexpected end of file |
| E013 | Parser | Mismatched indentation |
| E020 | Symbol | Unresolved variable `$X` |
| E021 | Symbol | Variable `$X` used before its declaring step |
| E022 | Symbol | Duplicate step name `X` in pipeline |
| E030 | Type | Type mismatch: expected `X`, got `Y` |
| E031 | Type | Field `X` does not exist on type `Y` |
| E032 | Type | Tool `X` is not registered |
| E033 | Type | MCP tool `X` schema fetch failed |
| W001 | Warning | Back-edge detected: `StepA → StepB` may cause infinite loop |
| W002 | Warning | Loop variable `$X` shadows pipeline-scope variable |
| W003 | Warning | Context budget exceeded at `StepX` (estimated `N` tokens) |
| W004 | Warning | No output declared for LLM call in `StepX` — result discarded |

---

## 7. Reserved Step Names

The following step names are reserved and have special semantics:

| Name | Meaning |
|---|---|
| `Done` | Terminal success step. Pipeline exits with code 0. |
| `Abort` | Terminal failure step. Pipeline exits with code 1. |
| `Error` | Terminal error step. Triggered on unhandled runtime errors. |

---

## 8. Changelog

| Version | Date | Change |
|---|---|---|
| 0.1.0 | 2026-06-28 | Initial specification |

---

## Related

- [Runtime Spec](runtime-spec.md) — how compiled programs execute
- [RFC-001](../03-rfc/rfc-001-agent-harness.md) — the architecture that motivated this spec
- [Compiler Design](../06-design/compiler-design.md) — implementation of this spec
