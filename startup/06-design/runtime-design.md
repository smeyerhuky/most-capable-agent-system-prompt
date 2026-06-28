---
type: Design Doc
title: "AgentForge Runtime Design"
description: Implementation design for the AgentForge runtime service. Covers the IR walker, model router, tool executor, context budget manager, checkpoint system, and event emission architecture.
tags: [design, runtime, ir-walker, model-router, tool-executor, events, checkpoint]
timestamp: 2026-06-28T00:00:00Z
---

# AgentForge Runtime Design

The runtime is an async Python service that walks compiled IR and dispatches instructions to model adapters, tools, and the event stream. It is the "VM" in the compiler pipeline analogy.

---

## Module Structure

```
agentforge/runtime/
├── runner.py          # Top-level: accepts a run_id, drives the IR walk loop
├── executor.py        # IR walker: instruction dispatch loop
├── context.py         # ExecutionContext: state bag for a single run
├── model_router.py    # Resolves model aliases → adapters, dispatches LLM calls
├── adapters/
│   ├── bedrock.py     # AWS Bedrock adapter (Claude, Llama 3)
│   ├── openai.py      # OpenAI adapter (GPT-4o)
│   └── gemma_vllm.py  # vLLM adapter (Gemma 4 private tier)
├── tools/
│   ├── registry.py    # ToolRegistry: maps tool names to ToolFn implementations
│   ├── fs.py          # fs.glob, fs.read, fs.write
│   ├── shell.py       # shell.run
│   ├── human.py       # human.checkpoint
│   └── retrieve.py    # retrieve()
├── context_budget.py  # Token counting, context buffer management
├── events.py          # ExecutionEvent types + Redis pub/sub emission
└── state.py           # ExecutionState persistence to Redis
```

---

## Execution Loop (`executor.py`)

```python
async def execute(ctx: ExecutionContext) -> None:
    while ctx.ip < len(ctx.ir.instructions):
        instr = ctx.ir.instructions[ctx.ip]
        
        try:
            await dispatch(instr, ctx)
        except ToolError as e:
            await ctx.emit(ToolCallFailed(tool=instr.target, error=str(e)))
            ctx.status = 'failed'
            return
        except ModelError as e:
            await ctx.emit(LLMCallFailed(model=instr.model, error=str(e)))
            ctx.status = 'failed'
            return
        
        await ctx.save_state()   # persist to Redis after every instruction
        
        if ctx.status == 'paused':
            return   # await external resume event
        
        ctx.ip += 1
```

**Instruction dispatch (`dispatch`):** A match/case over `instr.op`:

- `CALL` → `await tools.execute(instr.target, instr.args, ctx)`
- `LLM_CALL` → `await model_router.call(instr, ctx)`
- `ASSIGN` → `ctx.variables[instr.var] = ctx.resolve(instr.src)`
- `CONTEXT_PUSH` → `ctx.context_budget.push(instr.items, instr.budget)`
- `RETRIEVE` → loads file from S3 knowledge bundle
- `BRANCH` → evaluates condition, updates `ctx.ip` directly (no `+= 1`)
- `LOOP_START` → pushes loop frame onto `ctx.loop_stack`
- `LOOP_END` → checks loop frame, updates `ctx.ip` (jump or continue)
- `CHECKPOINT` → emits `CheckpointPaused`, sets `ctx.status = 'paused'`, returns
- `EMIT` → evaluates template, emits `RunCompleted` with message

---

## Execution Context (`context.py`)

```python
@dataclass
class ExecutionContext:
    run_id: str
    ir: IRDocument
    ip: int
    variables: dict[str, TypedValue]
    registers: dict[str, TypedValue]
    context_buffers: dict[str, ContextBuffer]
    loop_stack: list[LoopFrame]
    status: RunStatus
    event_channel: str          # Redis channel name for this run
    model_router: ModelRouter
    tool_registry: ToolRegistry
    
    def resolve(self, ref: str) -> TypedValue:
        # Handles $var, tmp0, $var.field, etc.
        ...
    
    async def emit(self, event: ExecutionEvent) -> None:
        await redis.publish(self.event_channel, event.to_json())
        await append_event_to_postgres(self.run_id, event)
    
    async def save_state(self) -> None:
        await redis.set(f'run:{self.run_id}:state', self.to_json(), ex=86400)
```

---

## Model Router (`model_router.py`)

```python
class ModelRouter:
    def __init__(self, aliases: dict[str, str], adapters: dict[str, ModelAdapter]):
        self.aliases = aliases      # e.g., {"default": "claude-3-5-sonnet-20241022"}
        self.adapters = adapters    # keyed by canonical model ID or prefix
        self.fallback = adapters['bedrock-llama3']
    
    async def call(self, instr: LLMCallInstruction, ctx: ExecutionContext) -> str:
        model_id = self.aliases.get(instr.model, instr.model)
        adapter = self.resolve_adapter(model_id)
        
        context_buffer = ctx.context_buffers.get(instr.context_register, [])
        prompt = self.render_template(instr.prompt, ctx.variables)
        
        await ctx.emit(LLMCallStarted(model=model_id, ...))
        
        try:
            response = await adapter.call(
                prompt=prompt,
                context=context_buffer,
                output_type=instr.output_type
            )
        except ModelAPIError as e:
            if adapter != self.fallback:
                # Retry once with fallback
                response = await self.fallback.call(prompt, context_buffer)
            else:
                raise
        
        await ctx.emit(LLMCallCompleted(model=model_id, tokens=response.tokens, ...))
        ctx.registers[instr.result] = TypedValue(response.content, instr.output_type)
        return response.content
    
    def resolve_adapter(self, model_id: str) -> ModelAdapter:
        if model_id.startswith('claude-'):
            return self.adapters['bedrock']
        if model_id.startswith('gpt-'):
            return self.adapters['openai']
        if model_id.startswith('gemma4-'):
            return self.adapters['gemma_vllm']
        if model_id.startswith('llama-'):
            return self.adapters['bedrock']
        raise UnknownModelError(model_id)
```

---

## Bedrock Adapter (`adapters/bedrock.py`)

```python
class BedrockAdapter(ModelAdapter):
    def __init__(self, region: str = 'us-east-1'):
        self.client = boto3.client('bedrock-runtime', region_name=region)
    
    async def call(self, prompt: str, context: list[str], output_type: str) -> ModelResponse:
        system = self.build_system_prompt(context)
        body = {
            "anthropic_version": "bedrock-2023-05-31",
            "max_tokens": 4096,
            "system": system,
            "messages": [{"role": "user", "content": prompt}]
        }
        
        start = time.monotonic()
        response = await asyncio.get_event_loop().run_in_executor(
            None,
            lambda: self.client.invoke_model_with_response_stream(
                modelId=self.model_id,
                body=json.dumps(body)
            )
        )
        
        content = self.stream_response(response)
        elapsed = time.monotonic() - start
        
        return ModelResponse(
            content=content,
            tokens=TokenCount(input=..., output=...),
            elapsed_ms=elapsed * 1000
        )
```

---

## Context Budget Manager (`context_budget.py`)

```python
class ContextBuffer:
    def __init__(self, budget: int):
        self.budget = budget
        self.items: list[tuple[str, int]] = []   # (content, token_count)
        self.total_tokens = 0
    
    def push(self, item: str, tokenizer: Tokenizer) -> PushResult:
        count = tokenizer.count(item)
        
        if self.total_tokens + count > self.budget:
            # Truncate oldest items to make room
            while self.items and self.total_tokens + count > self.budget:
                removed = self.items.pop(0)
                self.total_tokens -= removed[1]
            # Still over budget after truncation? Drop the new item
            if self.total_tokens + count > self.budget:
                return PushResult(pushed=False, truncated=True, warning=W003)
        
        self.items.append((item, count))
        self.total_tokens += count
        return PushResult(pushed=True, truncated=False)
    
    def render(self) -> str:
        return '\n\n'.join(content for content, _ in self.items)
    
    def reset(self):
        self.items.clear()
        self.total_tokens = 0
```

---

## Checkpoint System (`tools/human.py`)

The checkpoint system is the mechanism for human-in-the-loop workflows:

1. Runtime reaches `CHECKPOINT` instruction
2. `human.checkpoint()` tool is called with `data_ref` value
3. Tool serializes data to JSON, emits `CheckpointPaused` event to Redis
4. Runtime sets `ctx.status = 'paused'` and returns from execute loop
5. Run state is persisted to Redis
6. User sees the approval dialog in the web IDE or VS Code panel
7. User clicks Approve/Reject → `POST /runs/{id}/resume`
8. API Server publishes `ResumeEvent` to Redis channel for run
9. Runtime worker re-subscribes, fetches state from Redis, calls `execute()` again from `ip + 1`

In non-interactive mode (CI/CD), the `human.checkpoint` tool has a configurable timeout (default 300 seconds). If no resume event is received before timeout, the tool returns `ApprovalResult(status='rejected', note='timeout')`.

---

## Run Recovery

The runtime saves state after every instruction. If the runtime service restarts mid-run (e.g., ECS task replacement):

1. An orphan-run watcher polls Redis for runs in `running` status with no heartbeat (updated > 60 seconds ago)
2. For each orphan, the watcher dispatches the run to a new runtime worker
3. The worker loads `ExecutionState` from Redis and resumes from `ip`
4. The resumed execution re-emits any events from `ip` onward (clients deduplicate by event index)

---

## Related

- [Runtime Spec](../05-specs/runtime-spec.md) — the formal contract this design implements
- [GPU Serving Design](gpu-serving-design.md) — the Gemma vLLM adapter target
- [Compiler Design](compiler-design.md) — produces the IR consumed by this runtime
- [System Architecture](system-architecture.md) — where the runtime fits in the overall system
