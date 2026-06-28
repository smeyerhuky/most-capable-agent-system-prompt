---
type: Design Doc
title: "AgentForge GPU Serving Design"
description: Design for the AgentForge private GPU tier. Covers vLLM deployment of Gemma 4-26B-A4B with MTP speculative decoding on AWS g5.xlarge spot instances, including quantization configuration, auto-scaling, spot preemption handling, and multi-region expansion.
tags: [design, gpu, vllm, gemma4, mtp, speculative-decoding, aws, spot-instances]
timestamp: 2026-06-28T00:00:00Z
---

# AgentForge GPU Serving Design

---

## Why This Stack

The private GPU tier is the cost-competitive moat of AgentForge. The selection logic:

- **Gemma 4-26B-A4B (MoE):** 26B total parameters, 4B active per token. Quality close to 31B dense models at the inference cost of a 4B model. The right architecture for cost-efficient serving.
- **AWQ quantization:** 4-bit weights reduce VRAM from ~40GB (BF16) to ~12GB on A10G. Negligible quality degradation (<1% on coding benchmarks).
- **MTP speculative decoding:** The `-assistant` drafter model generates N candidate tokens; the target verifies them in one forward pass. With the Gemma 4 drafter (trained on the target's activations), acceptance rates of 60–75% are achievable, yielding 2–3× throughput improvement.
- **vLLM:** Production-grade inference server. Native support for speculative decoding, OpenAI-compatible API, continuous batching, and PagedAttention for memory efficiency.
- **AWS `g5.xlarge`:** A10G 24GB GPU, 4 vCPU, 16GB RAM. Spot price ~$0.35/hr. Fits the 26B-A4B Q4 model + assistant comfortably.

---

## Target Performance (A10G with Gemma 4-26B-A4B Q4 + MTP)

| Metric | Target | Measurement Method |
|---|---|---|
| Output tokens/sec | ≥ 50 | vLLM `/metrics` endpoint |
| Time to first token (TTFT) | ≤ 500ms | Per-request timing |
| MTP acceptance rate | ≥ 60% | vLLM speculative decoding stats |
| VRAM usage | ≤ 22GB | `nvidia-smi` |
| Concurrent requests handled | ≥ 20 | Load test with k6 |

---

## vLLM Configuration

### Launch Command

```bash
vllm serve google/gemma-4-26B-A4B-it \
  --speculative-model google/gemma-4-26B-A4B-it-assistant \
  --num-speculative-tokens 5 \
  --speculative-draft-tensor-parallel-size 1 \
  --quantization awq \
  --dtype bfloat16 \
  --max-model-len 32768 \
  --tensor-parallel-size 1 \
  --gpu-memory-utilization 0.92 \
  --host 0.0.0.0 \
  --port 8000 \
  --served-model-name gemma4-26b-private \
  --enable-prefix-caching \
  --disable-log-requests
```

### Key Configuration Notes

- `--num-speculative-tokens 5`: Draft 5 tokens at a time. The heuristic scheduler adjusts this dynamically (5 is the starting point; it goes up to 7 when acceptance is high, down to 3 when acceptance drops).
- `--enable-prefix-caching`: Caches KV values for shared prompt prefixes across requests. Critical for AgentForge because the system prompt and context window often share long prefixes between loop iterations.
- `--gpu-memory-utilization 0.92`: Leaves 8% VRAM headroom for the OS and CUDA overhead.
- `--max-model-len 32768`: Matches the default context budget in `.agent` pipelines.

### vLLM as OpenAI-Compatible API

vLLM exposes `POST /v1/chat/completions` with the same schema as OpenAI. The `GemmaVLLMAdapter` in the runtime is a thin wrapper:

```python
class GemmaVLLMAdapter(ModelAdapter):
    def __init__(self, base_url: str):
        self.client = AsyncOpenAI(base_url=base_url, api_key='not-used')
    
    async def call(self, prompt: str, context: list[str], output_type: str) -> ModelResponse:
        response = await self.client.chat.completions.create(
            model='gemma4-26b-private',
            messages=[
                {'role': 'system', 'content': self.build_context(context)},
                {'role': 'user', 'content': prompt}
            ],
            stream=True,
            max_tokens=4096
        )
        # stream and collect...
```

---

## AWS Infrastructure

### EC2 Setup (Phase 3A)

```
GPU Fleet:
  Instance type:  g5.xlarge (A10G 24GB)
  Capacity type:  Spot (primary), On-Demand (fallback)
  AMI:            Deep Learning AMI (CUDA 12.x, Ubuntu 22.04)
  EBS:            200GB gp3 (model weights + swap)
  
Networking:
  VPC:            private subnet (no public IP)
  Access:         Internal ALB in the same VPC
  Security group: Allow TCP 8000 from ALB SG only

Auto Scaling Group:
  Min capacity:   1 (always warm)
  Max capacity:   10
  Scale-out:      GPU util > 80% for 5 min → add 1 instance
  Scale-in:       GPU util < 20% for 15 min → remove 1 instance
  Warmup:         300 seconds (model load time on cold start)
```

### Spot Preemption Handling

AWS sends a 2-minute termination notice via the EC2 instance metadata endpoint before spot preemption. The GPU serving process monitors this:

```python
async def spot_termination_watcher():
    while True:
        resp = requests.get(
            'http://169.254.169.254/latest/meta-data/spot/termination-time',
            timeout=1
        )
        if resp.status_code == 200:
            # Termination notice received. Drain in-flight requests.
            await vllm_server.drain()
            # Notify runtime router to stop routing to this instance
            await redis.set(f'gpu:instance:{instance_id}:draining', '1', ex=300)
            break
        await asyncio.sleep(5)
```

When an instance drains, the runtime's `GemmaVLLMAdapter` falls back to Bedrock for the duration of the preemption. A new spot instance launches and warms up within 5 minutes.

---

## Model Weights Caching

Downloading `google/gemma-4-26B-A4B-it` (AWQ, ~12GB) on every instance start would take 15–20 minutes. To achieve < 5-minute cold start:

1. Pre-download weights to an EBS snapshot (AMI with weights baked in)
2. Launch new instances from this AMI → weights available immediately
3. Update the AMI when model versions change (monthly or on new Gemma releases)

Alternatively: S3 model cache. Mount S3 bucket via `s5cmd` to `/model-weights/` on startup. Faster than re-download from HuggingFace; still slower than AMI bake. Use AMI bake for Phase 3A simplicity.

---

## Monitoring

| Metric | Source | Alert Threshold |
|---|---|---|
| `vllm:tokens_per_second` | vLLM `/metrics` (Prometheus) | < 20 tok/s for 5 min → page |
| `vllm:mtp_acceptance_rate` | vLLM `/metrics` | < 40% for 10 min → investigate |
| `gpu:memory_used_pct` | `nvidia-smi` → CloudWatch | > 95% → OOM risk |
| `gpu:instance_count` | ASG → CloudWatch | 0 instances → critical alert |
| `gpu:queue_depth` | Runtime Redis queue | > 50 pending LLM calls → scale out |

Grafana dashboard: all GPU metrics in one view with time-series, overlaid with Bedrock fallback rate.

---

## Phase 3A Deployment Checklist

1. Request `g5.xlarge` spot quota in us-east-1 (default quota may be 0 — submit quota increase 2 weeks ahead)
2. Create AMI with CUDA + vLLM + baked model weights
3. Create Launch Template referencing the AMI
4. Create Auto Scaling Group with spot/on-demand mix (80% spot, 20% on-demand)
5. Create Internal ALB with health check on `/health` (vLLM returns 200 when ready)
6. Configure Runtime Service with `GEMMA_VLLM_URL` env var pointing to internal ALB DNS
7. Implement spot termination watcher as a sidecar process on each GPU instance
8. Add CloudWatch alarms → SNS → PagerDuty

---

## Multi-Region Expansion (Phase 3+)

Phase 3A: single-region (us-east-1) GPU fleet.

Phase 4 expansion:
- eu-west-1: L4 GPU on GCP (slightly cheaper for European latency)
- ap-southeast-1: Azure NC v4 (Azure's A100 offering has good APAC coverage)

Runtime router enhancement: route GPU tier requests to the nearest healthy GPU fleet based on the user's geographic region (stored on the user record).

---

## Related

- [System Architecture](system-architecture.md) — where the GPU fleet fits in the overall system
- [Runtime Design](runtime-design.md) — the `GemmaVLLMAdapter` that calls this fleet
- [PRD Phase 3A tasks](../02-prd/phase-3-scale.md#milestone-3a-private-gpu-tier-week-1720)
- [Revenue Model](../07-business-plan/revenue-model.md) — GPU tier billing and cost structure
