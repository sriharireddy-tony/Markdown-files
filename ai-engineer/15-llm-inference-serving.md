# 15 — LLM Inference & Serving

**What interviewers probe here:** whether you understand what actually happens on the GPU (prefill vs decode, KV cache, batching), can tune a serving engine, scale it on Kubernetes, and turn "10,000 users" into GPUs and dollars. Strong candidates explain *why* a metric or flag matters, not just its name.

[← Back to index](README.md)

1. [Walk through how you built an LLM inference microservice.](#q1)
2. [What inference optimization techniques do you know?](#q2)
3. [How does the KV cache work, and how is it managed?](#q3)
4. [Explain serving parameters: max number of sequences, max batched tokens, GPU memory utilization, max model length.](#q4)
5. [How do you set up HPA for an LLM service and choose its scaling signals?](#q5)
6. [Which latency metrics do you measure, and why?](#q6)
7. [How do you monitor P50/P95/P99 latency?](#q7)
8. [Which frameworks do you use for LLM deployment?](#q8)
9. [Stream tokens live over SSE. SSE vs WebSocket, and how do you scale WebSockets?](#q9)
10. [Size the hardware and cost for 10,000 real users.](#q10)
11. [The model doesn't fit your GPU. What are your options?](#q11)
12. [Containerize and deploy to a public URL: what does production-ready packaging look like?](#q12)
13. [Design a voice agent that must reply within ~1–1.2 s. What is the latency budget?](#q13)

---

<a id="q1"></a>
## Q1. Walk through how you built an LLM inference microservice.
*Source: interview posts*

**What the interviewer is testing:** Real ownership of the architecture, and the reasons behind each layer.

**Strong answer:**

"I'll go request path first, then the ops pieces.

```
Client ─► API Gateway (auth, rate limit, tenant id)
            │
            ▼
        FastAPI service  (validation, prompt templating, streaming, timeouts)
            │  OpenAI-compatible HTTP / gRPC
            ▼
        vLLM engine pods on GPU nodes (continuous batching, prefix cache)
            │
            ▼
        Model weights (quantized AWQ / FP8) from object storage → local NVMe cache
Side: Prometheus metrics, OpenTelemetry traces, structured logs, Redis cache
```

**Design decisions and why:**
- **Split the API layer from the engine.** The FastAPI layer is CPU-only and scales cheaply. It handles auth, request validation (max tokens, allowed params), prompt templates and response post-processing. The GPU engine does only inference. This keeps expensive GPU pods simple, and lets me swap engines.
- **vLLM** for continuous batching and PagedAttention, with an OpenAI-compatible API so clients don't change when I switch between self-hosted and a vendor.
- **Streaming via SSE** to cut perceived latency: time to first token matters more to users than total time.
- **Guardrails on requests:** cap `max_tokens`, cap input length, a per-tenant concurrency limit, and request timeouts with cancellation propagated to the engine, so abandoned requests free KV cache.
- **Model loading:** weights baked into or pre-pulled to node-local NVMe, since cold-pulling 16–40 GB from S3 on every pod start kills scale-up time. Readiness probe only after warm-up.
- **Observability:** TTFT, TPOT, queue time, tokens/s, KV-cache usage, error rate, all by model and tenant.
- **Scaling:** HPA on queue depth / running requests, not CPU (see Q5).

Results (illustrative): Llama-3-8B AWQ on L4s, P95 TTFT around 400 ms at 60 concurrent streams. Switching from naive HF `generate()` to vLLM gave about 8× throughput on the same GPUs."

**Follow-ups:**
- *Why not put everything in one container?* → You can, but independent scaling and failure isolation are the reasons to separate the API from the GPU engine.
- *How do you handle client disconnects?* → Detect the disconnect in the streaming loop and abort the request in the engine.
- *How do you version models?* → Model name + weights hash in config, with canary deployment by routing a percentage of traffic.

**Red flags:**
- "I used Flask and `model.generate()` per request." (No batching.)
- Not knowing what the service's P95 latency or throughput was.
- CPU-based autoscaling for a GPU service.

---

<a id="q2"></a>
## Q2. What inference optimization techniques do you know (continuous batching, PagedAttention, prefix caching, speculative decoding, FlashAttention, quantization)?
*Source: interview posts*

**What the interviewer is testing:** Whether you can map techniques to the bottleneck they fix.

**Strong answer:**

"First, the mental model. Inference has two phases:
- **Prefill:** process all prompt tokens in parallel. Compute-bound. Drives TTFT.
- **Decode:** generate one token at a time per sequence, reading all weights and the KV cache each step. Memory-bandwidth-bound. Drives TPOT and throughput.

| Technique | Fixes | How |
|---|---|---|
| **Continuous (in-flight) batching** | Low GPU utilization | New requests join the batch at every decode step instead of waiting for the whole batch to finish |
| **PagedAttention** | KV-cache fragmentation | KV cache stored in fixed-size blocks like OS pages. Near-zero waste, so more concurrent sequences |
| **Prefix caching** | Repeated system prompts / RAG templates | Reuse KV blocks for identical prefixes. Cuts TTFT and prefill compute |
| **Chunked prefill** | Long prompts stalling decode | Split prefill into chunks interleaved with decode steps. Smoother TPOT |
| **Speculative decoding** | Slow sequential decode | A small draft model (or n-gram / Medusa / EAGLE heads) proposes k tokens; the big model verifies them in one pass. Same output distribution, 1.5–3× faster when acceptance is high |
| **FlashAttention** | Attention memory traffic | Fused, tiled attention kernel. Avoids materializing the N×N matrix, so it's faster and uses less memory |
| **Quantization (AWQ/GPTQ/FP8, KV FP8)** | Memory and bandwidth | Fewer bytes per weight means faster decode and more room for KV cache |
| **Tensor / pipeline parallelism** | Model too big / latency | Split the model across GPUs |
| **Output length control** | Cost and latency | Output tokens dominate latency: cap `max_tokens`, ask for concise output |

**Order I'd apply them:** use a real serving engine (continuous batching + PagedAttention come free) → enable prefix caching, and structure prompts so the static part comes first → quantize (FP8 on H100, AWQ on smaller GPUs) → tune batch parameters → speculative decoding if the latency SLO still isn't met at low concurrency."

**Follow-ups:**
- *Does speculative decoding help at high batch sizes?* → Less. The GPU is already busy, so the verify overhead eats the gain. It shines at low-concurrency, latency-sensitive workloads.
- *Why is decode memory-bound?* → Each step reads every weight to produce one token per sequence. Arithmetic intensity is low unless the batch is large.

**Red flags:**
- Listing techniques without saying which bottleneck each fixes.
- "Just buy a bigger GPU."

---

<a id="q3"></a>
## Q3. How does the KV cache work, and how is it managed (memory math, paging, eviction)?
*Source: interview posts*

**What the interviewer is testing:** Transformer internals, and why the KV cache limits concurrency.

**Strong answer:**

"In attention, each new token's query attends over the keys and values of all previous tokens. Without a cache you'd recompute K and V for the whole sequence at every step, which is quadratic work. The **KV cache** stores each layer's K and V tensors for past tokens, so each decode step computes K and V only for the new token and appends them.

**Memory math (per token):**
```
KV bytes/token = 2 (K and V) × num_layers × num_kv_heads × head_dim × bytes_per_elem
```
Llama-3-8B (32 layers, 8 KV heads via GQA, head_dim 128, FP16):
`2 × 32 × 8 × 128 × 2 = 131,072 B ≈ 128 KB/token`
- One 8k-token sequence ≈ **1 GB**
- 64 concurrent 8k sequences ≈ **64 GB**, more than the whole model's weights.

Llama-3-70B (80 layers, 8 KV heads, 128): `2 × 80 × 8 × 128 × 2 ≈ 320 KB/token`, so 8k tokens ≈ 2.5 GB per sequence.

(Architecture figures are from the public configs; treat the totals as illustrative.)

So **KV cache, not weights, usually caps concurrency.** Ways it's managed:
- **GQA/MQA** (model design): fewer KV heads. Llama's 8 KV heads versus 32 query heads is a 4× reduction.
- **PagedAttention:** the cache is split into blocks (e.g. 16 tokens each) allocated on demand. There's no pre-reservation for max length, so fragmentation stays under a few percent, and prefixes can be shared copy-on-write.
- **Prefix caching:** identical prompt prefixes share blocks. LRU eviction of unreferenced blocks.
- **Preemption:** when the cache fills, the engine preempts a sequence and either swaps its blocks to CPU or recomputes later.
- **KV quantization (FP8):** halves the memory.
- **Limit `max-model-len`** to what the product needs. Serving 128k context when users send 4k wastes capacity planning headroom.
- **Offload / tiered caching** (CPU, SSD) for long-lived multi-turn sessions."

**Follow-ups:**
- *What happens when the KV cache is full?* → New requests queue; running ones may be preempted. You see TTFT spike and a 'num_preempted' metric rise.
- *Why does GQA barely hurt quality?* → Query heads share K/V projections. Empirically there's a small loss for a large memory win.
- *What is the KV cache's effect on cost?* → More cache means more concurrency per GPU, which means lower $/token.

**Red flags:**
- "The KV cache stores previous outputs as text."
- Not knowing it grows linearly with sequence length and batch size.

---

<a id="q4"></a>
## Q4. Explain serving parameters: max number of sequences, max batched tokens, GPU memory utilization, max model length.
*Source: interview posts*

**What the interviewer is testing:** Hands-on tuning of a serving engine (vLLM here).

**Strong answer:**

"These are the knobs that trade latency for throughput in vLLM:

| Flag | Meaning | Effect of raising it |
|---|---|---|
| `--max-num-seqs` | Max concurrent sequences in a batch (e.g. 256) | More throughput, but higher per-token latency (TPOT) and more KV pressure |
| `--max-num-batched-tokens` | Max tokens processed in one scheduler step (prefill + decode, with chunked prefill) | Larger means faster prefill of long prompts, but decode steps stall behind it, so TPOT jitters. Smaller means smoother streaming |
| `--gpu-memory-utilization` | Fraction of VRAM vLLM may use (e.g. 0.90). After loading weights, the rest becomes KV-cache blocks | More KV cache means more concurrency. Too high risks OOM with other processes or CUDA graphs |
| `--max-model-len` | Max context (prompt + output) per sequence | Lower frees scheduling headroom and startup profiling memory. Set it to what the product needs |
| `--enable-prefix-caching` | Reuse KV blocks for shared prefixes | Big TTFT wins for fixed system prompts and RAG templates |
| `--tensor-parallel-size` | Shard the model across N GPUs | Fits bigger models and lowers latency, with communication overhead |
| `--kv-cache-dtype fp8` | Quantize the KV cache | About 2× cache capacity |

**How I tune:**
1. Define the SLO, e.g. P95 TTFT < 800 ms and P95 TPOT < 50 ms at a target of 40 req/s.
2. Replay a **realistic traffic trace** (real prompt and output length distributions) with a load generator (vLLM's benchmark scripts, locust, genai-perf).
3. Sweep `max-num-seqs` (64/128/256) and `max-num-batched-tokens` (2k/8k/16k), and plot throughput vs P95 latency.
4. Pick the knee point: the most throughput that still meets the SLO.

Example: raising `max-num-seqs` from 64 to 256 roughly doubled tokens/s, but P95 TPOT went from 30 to 75 ms, which broke our streaming SLO. We settled at 128."

**Follow-ups:**
- *What do you watch in metrics?* → KV cache usage %, running vs waiting requests, preemptions, tokens/s.
- *Why does a long prompt hurt other users?* → Its prefill monopolizes a step. Chunked prefill plus a batched-token cap mitigate it.

**Red flags:**
- Setting `gpu-memory-utilization` to 0.99 "for max performance" without knowing what it controls.
- Tuning with synthetic fixed-length prompts only.

---

<a id="q5"></a>
## Q5. How do you set up HPA for an LLM service and choose its scaling signals?
*Source: interview posts*

**What the interviewer is testing:** Kubernetes autoscaling applied to GPU workloads.

**Strong answer:**

"Default HPA scales on CPU and memory, both of which are useless for LLM pods. GPU utilization is also misleading: it can read 100% while the service is healthy, or be at capacity while the metric looks fine. I scale on **load signals that track queueing**:

| Signal | Why |
|---|---|
| `vllm:num_requests_waiting` (queue depth) per pod | Direct sign that capacity is exhausted |
| `vllm:num_requests_running` / max-num-seqs | Saturation ratio |
| KV cache usage % | Near 90–100% means preemptions are coming |
| P95 TTFT | User-facing SLO, but lagging, so use it as a secondary signal |

**Setup:**
```
vLLM /metrics → Prometheus → Prometheus Adapter (or KEDA) → HPA custom metric
```
```yaml
metrics:
- type: Pods
  pods:
    metric: { name: vllm_num_requests_waiting }
    target: { type: AverageValue, averageValue: "5" }
behavior:
  scaleUp:   { stabilizationWindowSeconds: 30,  policies: [{type: Pods, value: 2, periodSeconds: 60}] }
  scaleDown: { stabilizationWindowSeconds: 600, policies: [{type: Pods, value: 1, periodSeconds: 300}] }
```

**Choosing the parameters:**
- **Target value:** load-test one pod to find the queue depth at which P95 TTFT breaches the SLO, then target about 60–70% of that.
- **Scale up fast, down slow.** GPU pods take 2–10 minutes to become ready (node provisioning, image pull, weight load, warm-up), so scale-down must be conservative to avoid flapping.
- **minReplicas ≥ 2** for availability. Keep warm capacity for predictable peaks, or schedule scale-ups ahead of known traffic patterns.
- **Cluster autoscaler / Karpenter** must provision GPU nodes too. HPA alone just creates pending pods.
- Speed up cold start: pre-pulled images, weights on local NVMe or a shared cache, readiness probe after a warm-up request.

KEDA is handy because it scales on Prometheus queries directly and supports scale-to-zero for dev environments."

**Follow-ups:**
- *Why not GPU utilization?* → The metric shows the fraction of time a kernel ran, not how much capacity is left. It saturates early and doesn't reflect queueing.
- *How do you handle burst traffic faster than scale-up?* → Queue with admission control, shed low-priority traffic, or burst overflow to an API provider.

**Red flags:**
- Scaling on CPU.
- Ignoring GPU node provisioning time and cold starts.

---

<a id="q6"></a>
## Q6. Which latency metrics do you measure (TTFT, TPOT/ITL, E2E, throughput), and why?
*Source: interview posts*

**What the interviewer is testing:** Whether you know which metric maps to which user experience and bottleneck.

**Strong answer:**

| Metric | Definition | Why it matters | Driven by |
|---|---|---|---|
| **TTFT** (time to first token) | Request arrival → first token | Perceived responsiveness in chat and voice | Queue time + prefill (prompt length), retrieval, guardrails |
| **TPOT / ITL** (time per output token / inter-token latency) | Average gap between tokens | Streaming smoothness; reading speed is ~5–10 tok/s | Decode, batch size, bandwidth |
| **E2E latency** | Request → last token | Matters for non-streamed / agent steps | TTFT + TPOT × output tokens |
| **Throughput** | Output tokens/s or requests/s per GPU | Cost efficiency | Batching, quantization |
| **Queue time** | Arrival → scheduled | Capacity signal for scaling | Load vs capacity |
| **Goodput** | Requests/s that *meet the SLO* | What you actually deliver | All of the above |

`E2E ≈ TTFT + TPOT × (output_tokens − 1)`

"For a chat product I'd set SLOs like P95 TTFT < 1 s and P95 TPOT < 60 ms. For a batch summarization job I'd ignore TTFT and optimize throughput and cost. In a RAG or agent system I also measure **per-stage latency**: embedding, vector search, reranker, LLM, tool calls. Users feel the whole pipeline, and often retrieval or a reranker is the slow part. Measure TTFT from the *client's* perspective too, since network and gateway time count."

**Follow-ups:**
- *Why does output length dominate?* → Each output token is a sequential decode step. 500 tokens × 40 ms = 20 s, while prefill of 2k tokens might take 200 ms.
- *Why goodput over throughput?* → High throughput with blown SLOs is useless to users.

**Red flags:**
- Only measuring average end-to-end latency.
- Not separating queue time from compute time.

---

<a id="q7"></a>
## Q7. How do you monitor P50/P95/P99 latency?
*Source: interview posts*

**What the interviewer is testing:** Observability mechanics and statistical sense.

**Strong answer:**

"**Collection:** emit latency as **histograms**, not averages. For example, a Prometheus histogram `llm_ttft_seconds` with buckets tuned for the range (0.05…10 s), labeled by model, route, tenant tier and streaming or not. Keep label cardinality bounded: no user IDs in labels. vLLM already exports TTFT, TPOT and E2E histograms.

**Computing percentiles:** `histogram_quantile(0.95, sum(rate(llm_ttft_seconds_bucket[5m])) by (le, model))`. Percentiles must be computed from the aggregated buckets. **You can't average P95s across pods.**

**Tracing:** OpenTelemetry spans for each stage (retrieve, rerank, LLM, tools), so when P99 spikes I can open exemplar traces and see which stage was slow.

**Why each percentile:**
- **P50:** typical experience; regressions show general slowdowns.
- **P95:** the SLO target; most users' worst typical case.
- **P99:** tail. Often caused by long prompts, preemption, cold pods, provider retries or GC pauses. At scale, 1% of 1M requests is 10k angry users, and agent chains amplify tails because one slow call in 10 makes the whole flow slow.

**Alerting:** on SLO burn rate (e.g. 'P95 TTFT > 1 s for 10 min' or error budget burn), not a single spike. Dashboards slice by prompt-length bucket, because a P99 driven by 30k-token prompts is a product question, not an infra one.

**Always pair latency with load** (req/s, tokens/s, queue depth) on the same dashboard so you can tell 'we're overloaded' from 'something is slow'."

**Follow-ups:**
- *P99 is spiking but P50 is fine. What do you check?* → Long-prompt outliers, preemptions, cold starts after scale-up, provider retries, one bad node.
- *Why not the mean?* → Latency is long-tailed. The mean hides the tail and is skewed by outliers.

**Red flags:**
- Averaging percentiles across instances.
- Only logging latencies without histograms or traces.

---

<a id="q8"></a>
## Q8. Which frameworks do you use for LLM deployment (vLLM, TGI, TensorRT-LLM, SGLang, Triton, Ollama)?
*Source: interview posts*

**What the interviewer is testing:** Awareness of the ecosystem and a reasoned choice.

**Strong answer:**

| Engine | Strengths | When I'd pick it |
|---|---|---|
| **vLLM** | PagedAttention, continuous batching, prefix caching, OpenAI-compatible API, broad model and hardware support, LoRA multi-adapter serving | Default for most self-hosted serving |
| **SGLang** | RadixAttention (strong prefix reuse), fast structured/constrained decoding, good for agent and multi-call programs | Heavy prefix sharing, structured output, agent workloads |
| **TensorRT-LLM** (+ Triton) | Highest raw performance on NVIDIA through compiled engines, FP8, in-flight batching | Max throughput on a fixed model at large scale, and you accept a build step |
| **Hugging Face TGI** | Mature, simple deployment, HF integration | Teams in the HF ecosystem |
| **Triton Inference Server** | Multi-model, multi-framework serving, ensembles | Mixing LLMs with other models (embeddings, classifiers) |
| **Ollama / llama.cpp** | Easy local, CPU/Mac, GGUF quantization | Local dev, edge, laptops. Not high-concurrency production |
| **Managed** (Bedrock, Vertex, Azure, SageMaker, Together, Fireworks) | No ops | When the team shouldn't run GPUs |

"My default is vLLM behind a thin FastAPI or gateway layer, because it gives 80–90% of the peak performance with the least friction, and its OpenAI-compatible API keeps clients portable. I'd consider TensorRT-LLM when GPU cost is large enough that another 20–30% efficiency justifies the build complexity. I always benchmark on *my* traffic shape before committing. Published benchmarks rarely match your prompt and output lengths."

**Follow-ups:**
- *How do you serve many fine-tunes cheaply?* → Multi-LoRA serving: one base model and many adapters loaded per request (vLLM/LoRAX-style).
- *Which do you use for embeddings?* → A lightweight server (TEI, Triton, or vLLM's embedding support) on smaller GPUs or CPUs.

**Red flags:**
- Only knowing Ollama.
- Picking an engine by popularity without benchmarking.

---

<a id="q9"></a>
## Q9. Stream tokens live over SSE. SSE vs WebSocket, and how do you scale WebSockets?
*Source: interview posts*

**What the interviewer is testing:** Practical streaming implementation and real-time scaling know-how.

**Strong answer:**

"**SSE (Server-Sent Events)** is one-way server→client streaming over plain HTTP, using `text/event-stream`. It's perfect for token streaming: it works through most proxies and load balancers, has auto-reconnect in browsers, and it's what the OpenAI-style APIs use.

```python
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse
from openai import AsyncOpenAI
import json

app = FastAPI()
client = AsyncOpenAI(base_url="http://vllm:8000/v1", api_key="x")

@app.post("/chat")
async def chat(req: Request, body: dict):
    async def event_stream():
        stream = await client.chat.completions.create(
            model="llama-3-8b", messages=body["messages"],
            stream=True, max_tokens=512)
        try:
            async for chunk in stream:
                if await req.is_disconnected():   # free GPU/KV cache
                    await stream.close()
                    break
                delta = chunk.choices[0].delta.content or ""
                if delta:
                    yield f"data: {json.dumps({'token': delta})}\n\n"
            yield "data: [DONE]\n\n"
        except Exception:
            yield f"event: error\ndata: {json.dumps({'error': 'generation_failed'})}\n\n"

    return StreamingResponse(event_stream(), media_type="text/event-stream",
                             headers={"Cache-Control": "no-cache",
                                      "X-Accel-Buffering": "no"})  # disable nginx buffering
```

Gotchas: disable proxy buffering, send heartbeat comments (`: ping`) for idle timeouts, set LB idle timeouts above the maximum generation time, and propagate cancellation on disconnect.

**SSE vs WebSocket:**
| | SSE | WebSocket |
|---|---|---|
| Direction | Server → client | Bidirectional |
| Protocol | HTTP (easy with LBs, auth, HTTP/2) | Upgrade to WS, stateful connection |
| Use case | Chat token streaming | Voice agents (audio both ways), barge-in / interruption, collaborative or live agent UIs |

**Scaling WebSockets:**
- Connections are **stateful and long-lived**, so capacity is bounded by concurrent connections (memory, file descriptors), not RPS. Tune ulimits, use an async server (uvicorn / uvloop), and budget roughly 10k–50k connections per node depending on the payload.
- LB with WS support. Sticky sessions or connection-aware routing.
- **Don't store session state in the process:** keep it in Redis so any node can resume the conversation after a reconnect.
- **Pub/sub fan-out** (Redis Streams, NATS, Kafka) when the producer (LLM worker) and the connection node differ.
- **Graceful drain on deploys:** stop accepting connections, tell clients to reconnect, and have clients reconnect with exponential backoff plus jitter to avoid a thundering herd.
- Separate the connection tier (cheap CPU) from GPU inference workers, connected by a queue."

**Follow-ups:**
- *Why not WebSocket for chat?* → It adds stateful complexity for no gain when the client only sends one request per turn.
- *How do you resume a dropped stream?* → Event IDs plus `Last-Event-ID`, or buffer the generated tokens in Redis for replay.

**Red flags:**
- Not handling client disconnects, so GPUs keep generating into the void.
- Keeping WebSocket session state in process memory with no plan for reconnects.

---

<a id="q10"></a>
## Q10. Size the hardware and cost for 10,000 real users.
*Source: interview posts*

**What the interviewer is testing:** Capacity estimation from first principles, and a build-vs-API cost comparison.

**Strong answer:**

"I'll state the assumptions, compute, then sanity-check. All numbers are illustrative.

**Assumptions:**
- 10,000 DAU, 10 requests each per day → **100k requests/day**
- Peak hour carries 20% of the day's traffic → 20k/hour ≈ **5.6 req/s peak**
- Each request: 1,500 input tokens (RAG context), 300 output tokens
- Target: P95 TTFT < 1 s, streaming at ≥ 20 tok/s per user

**Load at peak:**
- Output: 5.6 × 300 ≈ **1,700 output tok/s**
- Prefill: 5.6 × 1,500 ≈ **8,400 input tok/s**
- Concurrency (Little's law): each request streams 300 tokens at about 25 tok/s ≈ 12 s, plus ~0.5 s TTFT → 5.6 × 12.5 ≈ **70 concurrent sequences**

**KV-cache check (Llama-3-8B, FP16 KV ≈ 128 KB/token):**
70 sequences × 1,800 tokens × 128 KB ≈ **16 GB** of KV cache. Easily fits on one 80 GB GPU next to the 16 GB of FP16 weights.

**Throughput check:** an 8B model on one H100 with vLLM can typically sustain a few thousand output tok/s at ~70 concurrency. 1,700 tok/s fits on **1 GPU**. Add **N+1 for HA, so 2 × H100** (or about 4 × L4/A10G with an AWQ-quantized model).

**Cost (self-hosted):** 2 × H100 at roughly $2.5–4/GPU-hour × 730 h ≈ **$3.6k–5.8k/month**, plus CPU services, vector DB, observability, and a significant **engineering/on-call cost**.

**Cost (API), monthly volume:** 3M requests → 4.5B input + 0.9B output tokens.
| Option | Price (illustrative, per 1M in/out) | Monthly |
|---|---|---|
| Small API model | $0.15 / $0.60 | ≈ $675 + $540 ≈ **$1.2k** |
| Mid model | $1 / $4 | ≈ $4.5k + $3.6k ≈ **$8k** |
| Frontier model | $3 / $15 | ≈ $13.5k + $13.5k ≈ **$27k** |

**Conclusion:** at 10k users, a **small API model is cheaper than self-hosting**. Self-hosting an 8B model wins over the mid and frontier tiers only if the 8B model actually meets the quality bar, and it wins decisively at 10–50× this volume or when data can't leave the VPC. Before choosing, I'd cut tokens: prompt caching for the static system prompt (often 50–90% off cached input), semantic caching of repeated questions, and trimming RAG context from 1,500 to about 800 tokens with a reranker. Those can halve the bill either way.

**Sanity checks:** load-test with the real prompt distribution, check P95 not just the average, and keep headroom for 2–3× traffic spikes."

**Follow-ups:**
- *What if they want a 70B model?* → FP16 weights ≈ 140 GB, so 2 × H100 with TP=2 just for one replica, or FP8 on 1–2 GPUs. KV is 320 KB/token. Roughly 3–4× the cost.
- *What if usage is spiky?* → Keep a baseline self-hosted and burst to an API, or queue batch traffic.

**Red flags:**
- Jumping to "we need 8 A100s" without any math.
- Forgetting HA, peak factor, or the engineering cost of self-hosting.
- Ignoring output tokens, which are usually the expensive and slow part.

---

<a id="q11"></a>
## Q11. The model doesn't fit your GPU. What are your options (GPU constraints)?
*Source: interview posts*

**What the interviewer is testing:** Your toolbox for resource constraints and the trade-offs of each option.

**Strong answer:**

"First compute the requirement: weights (params × bytes) + KV cache (per-token math × concurrency × context) + activations/overhead (~10–20%). A 70B FP16 model is 140 GB of weights alone and doesn't fit on one 80 GB card.

Options, roughly in order:
1. **Quantize:** FP8 (≈70 GB), INT4 AWQ/GPTQ (≈35–40 GB). Usually the first move. Evaluate quality per task slice.
2. **Reduce KV needs:** lower `max-model-len`, cap `max-num-seqs`, use an FP8 KV cache.
3. **Tensor parallelism:** shard across GPUs in one node (NVLink). TP=2 or 4 for 70B. Lower latency, but needs fast interconnect.
4. **Pipeline parallelism:** split layers across GPUs or nodes. Works over slower links, adds pipeline bubbles.
5. **CPU/disk offload** (llama.cpp, accelerate offload): it fits but is slow. Fine for dev or batch, not interactive.
6. **Smaller or distilled model:** often the right answer. An 8B fine-tuned model may meet the task quality bar.
7. **MoE awareness:** a mixture-of-experts model has low *active* parameters (fast compute) but all experts must still be in memory.
8. **Rent differently:** bigger-memory GPUs (H200/B200/MI300X with 141–192 GB), or a managed API for that model.

Example: our team had only 4 × 24 GB L4s. A 70B model at INT4 needs about 40 GB of weights, so TP=4 across the L4s fits with around 40 GB left for KV cache, but PCIe (no NVLink) made TP slow at about 15 tok/s. We benchmarked a fine-tuned 8B AWQ model per GPU instead: 4 independent replicas, 5× the throughput, within 1.5 points on our eval. We shipped that."

**Follow-ups:**
- *TP vs PP?* → TP within a node for latency. PP across nodes when the model doesn't fit one node.
- *Why is TP slow on PCIe?* → An all-reduce per layer needs high bandwidth, and PCIe bottlenecks it.

**Red flags:**
- "Just use CPU offload in production."
- Not accounting for KV cache in the memory estimate.

---

<a id="q12"></a>
## Q12. Containerize and deploy to a public URL: what does production-ready packaging look like?
*Source: interview posts*

**What the interviewer is testing:** Basic DevOps maturity for AI services.

**Strong answer:**

"**Container:**
```dockerfile
FROM python:3.12-slim AS base
ENV PYTHONDONTWRITEBYTECODE=1 PYTHONUNBUFFERED=1
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app/ app/
RUN useradd -m appuser && chown -R appuser /app
USER appuser
EXPOSE 8000
HEALTHCHECK CMD curl -f http://localhost:8000/healthz || exit 1
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "2"]
```
- Pin dependency versions. Use a slim, multi-stage image. Run as non-root.
- **No secrets in the image.** API keys come from env vars or a secrets manager at runtime.
- GPU images start from the vendor base (e.g. the official `vllm/vllm-openai` image). **Don't bake model weights into huge images** unless startup speed demands it. Mount them from a volume or cache, pinned by version or hash.

**Service readiness:**
- `/healthz` (liveness) vs `/readyz` (model loaded, dependencies reachable).
- Structured JSON logs, a Prometheus `/metrics` endpoint, trace propagation.
- Config via env vars: model name, timeouts, max tokens.
- Graceful shutdown: finish in-flight streams on SIGTERM.

**Deploy to a URL:**
- Simple path: Cloud Run / App Runner / Azure Container Apps / Fly.io / Render for the API, calling an LLM API.
- Scale path: Kubernetes (Deployment + Service + Ingress), TLS via cert-manager or the cloud LB, HPA.
- **Public-internet protections:** auth (API key/OAuth), per-user rate limiting, input size limits, CORS restrictions, WAF, and a **spend cap** on the LLM account, because an open LLM endpoint gets abused within hours.
- CI/CD: build → unit tests → **eval smoke test** (a small golden set must pass) → image scan → deploy to staging → canary → production."

**Follow-ups:**
- *Why an eval step in CI?* → A prompt or model change can pass unit tests and still break answer quality.
- *How do you roll back?* → Immutable image tags, and redeploy the previous tag. Prompts and config are versioned separately.

**Red flags:**
- Hard-coded API keys in the repo or image.
- A public endpoint with no auth, rate limit or spend limit.

---

<a id="q13"></a>
## Q13. Design a voice agent that must reply within ~1–1.2 s. What is the latency budget?
*[Added]*

**What the interviewer is testing:** End-to-end latency engineering across streaming components.

**Strong answer:**

"The budget is measured from **the user stopping speaking** to **the first audio the user hears**. Every stage has to stream and overlap.

```
User speech ─► VAD/endpointing ─► streaming ASR ─► [RAG + LLM] ─► streaming TTS ─► audio out
```

**Illustrative budget (target ≈ 1.0–1.2 s):**
| Stage | Budget | Techniques |
|---|---|---|
| End-of-turn detection | 200–300 ms | Tuned VAD silence threshold + semantic end-of-turn model; this is the biggest tunable |
| ASR final | 100–200 ms | Streaming ASR; partial transcripts arrive during speech |
| Retrieval | 50–150 ms | Start on partial transcripts; in-region vector DB; cache frequent intents |
| LLM TTFT | 250–400 ms | Fast model (Flash-class / small), prompt caching, short system prompt, same region |
| First sentence → TTS first audio | 100–200 ms | Stream LLM tokens into TTS at sentence/phrase boundaries |
| Network / telephony | 50–150 ms | Co-locate services, keep WebSocket connections warm |

**Key design points:**
- **Pipeline overlap:** never wait for full outputs. LLM tokens flow into TTS as soon as the first clause is ready.
- **But make sure retrieval actually completes.** A real incident pattern: to save latency, retrieval was fired asynchronously and the prompt was built before the results arrived, so the model answered from its priors and hallucinated facts. The fix: await retrieval with a hard timeout (e.g. 150 ms). On timeout, use a safe fallback answer ('let me check that for you'), never generation without context.
- **Short answers** (1–2 sentences) suit voice and cut TTS time.
- **Barge-in:** if the user interrupts, cancel TTS and LLM generation immediately, over a bidirectional WebSocket.
- **Filler/acknowledgment** audio for slow tool calls ('one moment…') to mask latency honestly.
- **Monitor per stage** at P50/P95, per language, since ASR and TTS latency differ a lot by language."

**Follow-ups:**
- *Where do you cut first if P95 is 1.8 s?* → Endpointing threshold, LLM model size and prompt length, and region co-location, in that order. Check the traces.
- *Speech-to-speech models?* → Lower latency and more natural, but less control over grounding, tools and guardrails. Evaluate them for your compliance needs.

**Red flags:**
- Treating the stages as sequential blocking calls.
- Sacrificing grounding for latency, with no timeout fallback.

---

## Rapid-fire recap

- Prefill is compute-bound (drives TTFT). Decode is bandwidth-bound (drives TPOT and throughput).
- KV bytes/token = 2 × layers × kv_heads × head_dim × bytes. Llama-3-8B ≈ 128 KB/token, so 8k tokens ≈ 1 GB.
- The KV cache, not weights, usually caps concurrency. PagedAttention, GQA, an FP8 KV cache and a sane `max-model-len` all help.
- Continuous batching plus prefix caching are the biggest free wins. Put the static prompt part first.
- `--max-num-seqs` and `--max-num-batched-tokens` trade throughput against TPOT. Tune on real traces to the SLO knee.
- Scale on queue depth / KV usage, not CPU or GPU%. Scale up fast, down slow, and plan for GPU cold starts.
- Use histograms for latency. Never average percentiles. Track P99 separately.
- vLLM is the default engine. TensorRT-LLM for max efficiency, SGLang for prefix-heavy and structured work.
- SSE for chat streaming, WebSocket for bidirectional voice. Always cancel on disconnect.
- Sizing: DAU → peak req/s → tokens/s → concurrency (Little's law) → KV memory → GPUs → $, then compare with the API.
- A voice budget of ~1 s requires overlapping streams, and awaiting retrieval with a timeout, never skipping it.
