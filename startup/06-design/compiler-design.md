---
type: Design Doc
title: "AgentForge Compiler Design"
description: Implementation design for the AgentForge .agent DSL compiler. Covers the lexer, recursive descent parser, AST construction, symbol table, type checker, and IR emitter. Maps directly to classical compiler theory (Ch. 3-8).
tags: [design, compiler, lexer, parser, ast, symbol-table, type-checker, ir]
timestamp: 2026-06-28T00:00:00Z
---

# AgentForge Compiler Design

The compiler implements the pipeline described in RFC-001 and the grammar defined in [language-spec.md](../05-specs/language-spec.md). It is a stateless service — any source file in yields the same IR out, regardless of environment.

---

## Module Structure

```
agentforge/compiler/
├── lexer.py         # Tokenizer
├── parser.py        # Recursive descent parser → ParseTree
├── ast_builder.py   # ParseTree → TaskGraph AST
├── symbols.py       # Symbol table construction and resolution
├── types.py         # Type definitions and inference rules
├── checker.py       # Type checker (validates AST against type rules)
├── ir_emitter.py    # TaskGraph → IR JSON
├── errors.py        # Error/warning types (ErrorCode enum, CompileError, Warning)
└── compiler.py      # Top-level: source → IR (or errors), ties all phases together
```

---

## Phase 1: Lexer (`lexer.py`)

**Class:** `Lexer(source: str)`

**Output:** `list[Token]`

**Token dataclass:**
```python
@dataclass
class Token:
    kind: TokenKind       # Enum: KEYWORD, IDENT, VAR, STRING, NUMBER, ...
    value: str            # Raw matched text
    line: int
    col: int
```

**Algorithm:** Single-pass character scan. State machine for:
- Indentation tracking: stack of indent levels, emit `INDENT`/`DEDENT` on changes
- String parsing: handle escape sequences and `{{ }}` template regions
- Comment stripping: consume from `#` to `\n`

**Error recovery:** On illegal character, emit `E001` and advance past the character (continue tokenizing — collect all errors in one pass).

---

## Phase 2: Parser (`parser.py`)

**Class:** `Parser(tokens: list[Token])`

**Output:** `ParseTree` (sum type tree of nodes)

**Algorithm:** Recursive descent. Each grammar production is a method:

```python
def parse_program(self) -> ProgramNode:
    pipelines = []
    while not self.at_end():
        pipelines.append(self.parse_pipeline())
    return ProgramNode(pipelines)

def parse_pipeline(self) -> PipelineNode:
    self.expect(KEYWORD, 'pipeline')
    name = self.expect(IDENT)
    self.expect(COLON)
    self.expect(NEWLINE)
    self.expect(INDENT)
    body = self.parse_pipeline_body()
    self.expect(DEDENT)
    return PipelineNode(name=name.value, body=body)
```

**Error recovery:** On unexpected token, emit `E010`/`E011` and skip to the next `NEWLINE` + `INDENT` boundary (synchronize to the next step or pipeline declaration).

---

## Phase 3: AST Builder (`ast_builder.py`)

**Class:** `ASTBuilder(parse_tree: ProgramNode)`

**Output:** `list[TaskGraph]`

Transforms `PipelineNode` → `TaskGraph`:

1. For each `StepNode` in pipeline body → `TaskNode`
2. Extract step-level options: `model`, `context`, `prompt`, `tool`, `loop`, `branch`, `condition`, `output`
3. Build edges: sequential steps get `control` edges; `branch` arms get `conditional` edges with labels; `loop` end connects back to loop start (back-edge)
4. Back-edge detection: DFS over edges; back-edges are flagged as `W001` warnings (not removed — loops are valid)

**TaskGraph definition:**
```python
@dataclass
class TaskGraph:
    name: str
    options: PipelineOptions
    nodes: list[TaskNode]
    edges: list[Edge]
    back_edges: list[Edge]   # detected cycles, for warning emission
```

---

## Phase 4: Symbol Table (`symbols.py`)

**Class:** `SymbolTable`

Built by walking the TaskGraph nodes in topological order (DFS from entry node).

**Rules:**
- A step's `output: $var` declaration creates a symbol entry `{ name, type: None (inferred later), defined_at_step: str }`
- Loop `for $item in $list` creates a symbol entry scoped to the loop body
- A reference to `$var` in a step that precedes `$var`'s declaration step emits `E021`
- A reference to an unknown `$var` emits `E020`
- Shadow detection: loop variable name matches a pipeline-scope variable → `W002`

**Serialization:** `SymbolTable.to_dict()` produces a JSON-serializable dict consumed by the LSP.

---

## Phase 5: Type Checker (`checker.py`)

**Class:** `TypeChecker(graph: TaskGraph, symbols: SymbolTable)`

Annotates each `TaskNode` with inferred types for its inputs and output. Validates:

1. Tool call argument types (e.g., `fs.read` expects `String | Item<FileList>`)
2. Variable pass-through types (edge data types match receiving step's context_refs)
3. Branch conditions: field access on correct type (e.g., `.exit_code` only on `ExecResult`)
4. Loop iterable types: `for $item in $list` where `$list` must be `List<T>` → `$item` is `T`

**Implementation pattern (example):**
```python
def check_tool_call(self, node: TaskNode) -> Type:
    tool = self.tool_registry.get(node.tool.name)
    if tool is None:
        self.emit_error(E032, node)
        return UnknownType
    for i, (arg, expected_type) in enumerate(zip(node.tool.args, tool.arg_types)):
        actual_type = self.resolve_type(arg)
        if not self.is_assignable(actual_type, expected_type):
            self.emit_error(E030, node, expected=expected_type, actual=actual_type)
    return tool.return_type
```

**Type inference for LLM calls:**
- Default return type: `MarkdownContent`
- If prompt ends with "Return JSON." and output declares `json_list`: `JsonList`
- If output declares `json`: `JsonObject`
- Future: structured output schemas (Phase 2+)

---

## Phase 6: IR Emitter (`ir_emitter.py`)

**Class:** `IREmitter(graph: TaskGraph, symbols: SymbolTable)`

**Output:** `IRDocument` (JSON-serializable)

Walks the TaskGraph in execution order (topological sort from entry node). For each `TaskNode`, emits a sequence of IR instructions:

```
TaskNode with context + LLM + output:
  → CONTEXT_PUSH instructions (one per context_ref)
  → LLM_CALL instruction
  → ASSIGN instruction

TaskNode with tool:
  → CALL instruction
  → ASSIGN instruction (if output declared)

TaskNode with loop:
  → LOOP_START
  → (loop body instructions, recursively emitted)
  → LOOP_END

TaskNode with branch:
  → BRANCH instruction (cases point to IR indices of target steps)
```

**Index assignment:** Each instruction is assigned a sequential `index` (0, 1, 2, ...). `BRANCH.cases.target` and `LOOP_END.start_target` are IR instruction indices, resolved during emission via a two-pass approach:
1. First pass: emit instructions with placeholder targets
2. Second pass: fill in instruction indices now that all positions are known

---

## Compiler Entry Point (`compiler.py`)

```python
def compile(source: str) -> CompileResult:
    tokens = Lexer(source).tokenize()
    if tokens.has_errors:
        return CompileResult(ok=False, errors=tokens.errors)
    
    parse_tree = Parser(tokens.tokens).parse()
    if parse_tree.has_errors:
        return CompileResult(ok=False, errors=parse_tree.errors)
    
    graphs = ASTBuilder(parse_tree.tree).build()
    
    all_errors, all_warnings = [], []
    all_irs = []
    
    for graph in graphs:
        symbols = SymbolTable(graph).build()
        all_errors.extend(symbols.errors)
        all_warnings.extend(symbols.warnings)
        
        if not symbols.errors:
            types = TypeChecker(graph, symbols).check()
            all_errors.extend(types.errors)
            all_warnings.extend(types.warnings)
            
            if not types.errors:
                ir = IREmitter(graph, symbols).emit()
                all_irs.append(ir)
    
    if all_errors:
        return CompileResult(ok=False, errors=all_errors, warnings=all_warnings)
    return CompileResult(ok=True, ir=all_irs, warnings=all_warnings)
```

---

## Testing Strategy

- **Lexer:** 50+ unit tests covering all token types, edge cases (empty file, tab indentation error, nested template expressions)
- **Parser:** 30+ unit tests covering all grammar productions, error recovery cases
- **AST Builder:** 20+ unit tests covering back-edge detection, option parsing, edge construction
- **Symbol Table:** 15+ tests covering all scoping rules, shadow detection
- **Type Checker:** 30+ tests covering all type rules, each error code
- **IR Emitter:** 10+ golden-file tests (source file → expected IR JSON, exact match)
- **Integration:** 5 end-to-end tests (full pipeline `.agent` files → IR → manually verified)

---

## Related

- [Language Spec](../05-specs/language-spec.md) — the grammar and types this compiler implements
- [Runtime Design](runtime-design.md) — the runtime that executes the IR this compiler produces
- [PRD Phase 1A tasks](../02-prd/phase-1-mvp.md#milestone-1a-compiler-core-week-1-2) — task decomposition
