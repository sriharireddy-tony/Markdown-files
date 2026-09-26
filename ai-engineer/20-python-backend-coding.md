# 20 — Python, Backend & Coding

**What interviewers probe here:** whether you can build the service around the model. That means concurrency, API design, database connections, rate limits and streaming, plus a live coding problem. AI engineers who can't write solid backend code struggle to ship production systems.

[← Back to index](README.md)

1. [Async: when and when not](#q1)
2. [Benefits of FastAPI](#q2)
3. [DB connection strings and secrets](#q3)
4. [DB connections for many users / SQLAlchemy pool](#q4)
5. [GIL, threads vs processes vs asyncio](#q5)
6. [Code: concurrent LLM calls with rate limit + backoff](#q6)
7. [Code: streaming from FastAPI](#q7)
8. [Coding: Maximum Subarray Sum (Kadane)](#q8)
9. [Coding: Reciprocal Rank Fusion](#q9)
10. [Coding: chunker with overlap + cosine top-k](#q10)
11. [HTTP fundamentals for AI engineers](#q11)
12. [Pydantic's role in GenAI apps](#q12)

---

<a id="q1"></a>
## Q1. When would you use asynchronous programming, and when not?
*Source: interview posts*

**What the interviewer is testing:** Whether you understand I/O-bound vs CPU-bound work and the event-loop blocking pitfall.

**Strong answer:**

Async (`asyncio`) lets one thread handle many tasks that spend most of their time **waiting**. When a coroutine hits `await` on I/O, the event loop runs something else.

**Use async when the work is I/O-bound and concurrent:**
- LLM API calls (1–30 s each, mostly waiting on the network)
- Fan-out: parallel retrieval from a vector DB, BM25 and a reranker service; parallel tool calls
- Streaming responses (SSE/WebSockets) with many open connections
- High-concurrency API servers (FastAPI)

Example: a RAG request that calls the embedding API (80 ms), vector search (40 ms) and keyword search (30 ms) sequentially takes about 150 ms. With `asyncio.gather` on the two searches it takes about 120 ms. At the server level, one async worker can hold hundreds of in-flight LLM calls instead of blocking a thread per request.

**Don't use async (or don't use it alone) when:**
- The work is **CPU-bound**: tokenization of big corpora, PDF parsing, embeddings computed locally on the CPU, pandas transforms. Async gives no speedup here, and it **blocks the event loop** so all other requests stall. Use a process pool, a separate worker service, or `run_in_executor`.
- Your libraries are synchronous only (some DB drivers, SDKs). Calling them inside `async def` blocks the loop. Use their async variants or offload them to threads.
- Simple scripts or batch jobs with no concurrency need; plain sync code is simpler to read and debug.

The classic bug: `time.sleep(2)` or `requests.get()` inside an `async def` FastAPI endpoint freezes the whole worker. Use `await asyncio.sleep` and `httpx.AsyncClient`, or declare the endpoint with plain `def` so FastAPI runs it in a threadpool.

**Follow-ups:**
- *`gather` vs `TaskGroup`?* → `TaskGroup` (3.11+) cancels sibling tasks on failure; it has structured concurrency semantics.
- *How do you bound concurrency?* → Use `asyncio.Semaphore` (see Q6).

**Red flags:**
- "Async makes everything faster."
- Sync HTTP calls inside async endpoints.

---

<a id="q2"></a>
## Q2. What are the benefits of FastAPI?
*Source: interview posts*

**What the interviewer is testing:** Why it's the default choice for AI services, and knowing its limits.

**Strong answer:**

- **Async-native (ASGI on Starlette/Uvicorn):** good for I/O-heavy LLM workloads and streaming (SSE, WebSockets).
- **Pydantic validation:** request and response models from type hints, automatic 422 errors, and the same models are reused for LLM structured output.
- **Automatic OpenAPI/Swagger docs:** easy for frontend teams and tool/agent integrations (OpenAPI specs can become tool definitions).
- **Dependency injection** (`Depends`): clean handling of auth, DB sessions, rate limiters and per-request context.
- **Performance:** among the faster Python frameworks. Realistically the LLM call dominates latency, so this matters less than people think.
- **Developer speed:** type hints give editor support and fewer bugs.
- **Built-ins:** background tasks, middleware, lifespan events (load models or clients once at startup).

Trade-offs: Python is still slow for CPU-heavy paths; mixing sync and async code incorrectly blocks the loop; and for heavy background work you still need a real queue (Celery/RQ/Arq), not `BackgroundTasks`.

```python
from fastapi import FastAPI, Depends
from pydantic import BaseModel

app = FastAPI()

class AskRequest(BaseModel):
    question: str
    top_k: int = 5

class AskResponse(BaseModel):
    answer: str
    sources: list[str]

@app.post("/ask", response_model=AskResponse)
async def ask(req: AskRequest, user=Depends(get_current_user)):
    ...
```

**Follow-ups:**
- *Flask vs FastAPI?* → Flask is WSGI and sync-first; FastAPI gives async, validation and docs out of the box.
- *Running in production?* → Uvicorn workers (via Gunicorn or directly) behind a load balancer; workers ≈ CPU cores for CPU work, and fewer workers suffice for async I/O.

**Red flags:**
- "It's fast because it's async", with no nuance about blocking.

---

<a id="q3"></a>
## Q3. How do you create a database connection string, and how do you manage secrets?
*Source: interview posts*

**What the interviewer is testing:** Basic correctness plus secret hygiene.

**Strong answer:**

The SQLAlchemy URL format:

```
dialect+driver://username:password@host:port/database?options
```

Examples:
- `postgresql+psycopg://app_user:***@db.internal:5432/ragdb?sslmode=require`
- `postgresql+asyncpg://app_user:***@db.internal:5432/ragdb` (async)
- `mysql+pymysql://user:***@host:3306/db`

Build it safely, because special characters in passwords (`@`, `/`, `#`) break naive string concatenation:

```python
from sqlalchemy import URL

url = URL.create(
    "postgresql+psycopg",
    username=settings.db_user,
    password=settings.db_password,   # escaped automatically
    host=settings.db_host,
    port=5432,
    database="ragdb",
    query={"sslmode": "require"},
)
```

Secrets management:
- Never hardcode credentials or commit `.env` files. Load them via environment variables using `pydantic-settings`.
- In production, use a secret store: Azure Key Vault, AWS Secrets Manager, HashiCorp Vault, or Kubernetes secrets with encryption. Better still, use **managed identity / IAM auth** so there's no static password.
- Rotate credentials; use separate users per service with least privilege (the RAG reader gets a read-only role).
- Mask connection strings in logs.

**Follow-ups:**
- *Why `sslmode=require`?* → It encrypts traffic in transit; `verify-full` also validates the server certificate.

**Red flags:**
- A password in a Git repo or in a Docker image layer.

---

<a id="q4"></a>
## Q4. How do you handle database connections for many users? What is SQLAlchemy's connection pool?
*Source: interview posts*

**What the interviewer is testing:** Understanding that connections are expensive and limited, and knowing how pooling works.

**Strong answer:**

Opening a DB connection costs a TCP handshake, TLS and auth, often 20–100 ms. Postgres also caps connections (`max_connections` is often 100–500), and each connection uses server memory. So you never open a connection per request; you **reuse** them from a pool.

SQLAlchemy's `QueuePool` (the default) keeps a set of open connections. A request checks one out and returns it when done.

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

engine = create_engine(
    url,
    pool_size=10,        # persistent connections kept open
    max_overflow=20,     # extra temporary connections under burst
    pool_timeout=30,     # seconds to wait for a free connection before erroring
    pool_recycle=1800,   # recycle connections older than 30 min (avoid server/LB idle kills)
    pool_pre_ping=True,  # test connection on checkout; replaces dead ones transparently
)
SessionLocal = sessionmaker(bind=engine)

def get_db():                    # FastAPI dependency: one session per request
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()               # returns connection to the pool
```

Sizing math: each app process has its own pool. With 4 pods × 4 workers × (10 + 20) you could open up to **480 connections**, which can exceed the DB limit. So:
- Size the pool per process from the concurrency actually needed.
- Put **PgBouncer** (transaction pooling) in front of Postgres for many pods.
- For async, use `create_async_engine` with asyncpg; the same pool concepts apply.

Also: keep transactions short, and **never hold a DB connection while waiting on an LLM call** (a 10-second hold per request exhausts the pool quickly). Fetch the data, release the connection, then call the model.

**Follow-ups:**
- *Symptoms of pool exhaustion?* → `TimeoutError: QueuePool limit ... reached`, and latency spikes under load.
- *Serverless?* → Use `NullPool` plus an external pooler (RDS Proxy / PgBouncer).

**Red flags:**
- Creating an engine inside the request handler.
- Not knowing the pool is per process.

---

<a id="q5"></a>
## Q5. GIL, threads vs processes vs asyncio for LLM workloads.
*[Added]*

**What the interviewer is testing:** Choosing the right concurrency model.

**Strong answer:**

The **GIL** (Global Interpreter Lock) in CPython lets only one thread execute Python bytecode at a time. Threads still help for I/O, because the GIL is released while waiting, and C extensions like NumPy and PyTorch release it during heavy computation. Python 3.13+ has an experimental free-threaded build, but most production code today assumes the GIL.

| Workload | Best tool | Why |
|---|---|---|
| Many LLM/API calls, DB queries | `asyncio` | Thousands of concurrent waits, low overhead |
| Blocking I/O libraries (sync SDKs) | `ThreadPoolExecutor` | Releases GIL on I/O |
| CPU-heavy pure Python (PDF parsing, regex over GBs) | `ProcessPoolExecutor` / multiprocessing / worker service | Bypasses GIL with separate processes |
| GPU inference | Dedicated serving process (vLLM/TGI) | Batching on GPU; the web app just calls it |
| Large batch ingestion | Queue + horizontally scaled workers (Celery, Ray) | Retries, scaling, isolation |

Example: an ingestion pipeline parsed 200k PDFs. Async did nothing for parsing, which is CPU-bound. A process pool of 16 workers cut the runtime from about 9 h to 45 min. Embedding API calls then used asyncio with a semaphore of 32.

**Follow-ups:**
- *Why not threads for everything?* → CPU-bound Python won't parallelize with threads, and thousands of threads are memory-heavy.

**Red flags:**
- "Use multithreading to speed up CPU-bound Python."

---

<a id="q6"></a>
## Q6. Code: call an LLM concurrently with a rate limit and retries with backoff.
*[Added]*

**What the interviewer is testing:** Production-grade async code: bounded concurrency, retrying the right errors, jitter and timeouts.

**Strong answer:**

```python
import asyncio
import random
import httpx

RETRYABLE = {429, 500, 502, 503, 504}

async def call_llm(client: httpx.AsyncClient, sem: asyncio.Semaphore,
                   prompt: str, max_retries: int = 5) -> str:
    async with sem:                                   # bound concurrency
        for attempt in range(max_retries + 1):
            try:
                r = await client.post("/v1/chat", json={"prompt": prompt}, timeout=30.0)
                if r.status_code in RETRYABLE:
                    raise httpx.HTTPStatusError("retryable", request=r.request, response=r)
                r.raise_for_status()                  # 4xx (except 429) -> fail fast
                return r.json()["text"]
            except (httpx.TimeoutException, httpx.TransportError, httpx.HTTPStatusError) as e:
                status = getattr(getattr(e, "response", None), "status_code", None)
                if status is not None and status not in RETRYABLE:
                    raise
                if attempt == max_retries:
                    raise
                retry_after = None
                if status == 429:
                    ra = e.response.headers.get("retry-after")
                    retry_after = float(ra) if ra and ra.isdigit() else None
                # exponential backoff with full jitter, capped at 30s
                delay = retry_after or random.uniform(0, min(30, 0.5 * 2 ** attempt))
                await asyncio.sleep(delay)

async def run_batch(prompts: list[str]) -> list[str | BaseException]:
    sem = asyncio.Semaphore(16)
    async with httpx.AsyncClient(base_url="https://llm.internal") as client:
        return await asyncio.gather(*(call_llm(client, sem, p) for p in prompts),
                                    return_exceptions=True)
```

Talking points:
- **Semaphore** caps in-flight requests; for token-per-minute limits, add a token-bucket limiter keyed on estimated tokens.
- **Retry only transient errors** (429, 5xx, timeouts). A 400 means your request is wrong, and retrying it wastes money.
- **Full jitter** avoids a thundering herd where every client retries at the same moment.
- Honour `Retry-After`.
- `return_exceptions=True` so one failure doesn't kill the batch; log the failures and send them to a dead-letter list.
- Retrying calls that have **side effects** requires idempotency keys.

**Follow-ups:**
- *Libraries?* → `tenacity` for retry policies, `aiolimiter` for rate limits; the SDKs of major providers also have built-in retries.
- *Circuit breaker?* → After N consecutive failures, stop calling the provider and fail over for a cool-down period.

**Red flags:**
- An unbounded `gather` over 10k prompts.
- Fixed-interval retries without jitter.

---

<a id="q7"></a>
## Q7. Code: stream an LLM response from FastAPI.
*[Added]*

**What the interviewer is testing:** SSE formatting, async generators, disconnect handling.

**Strong answer:**

```python
import json
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse

app = FastAPI()

async def token_stream(prompt: str):
    # e.g. provider SDK streaming: async for chunk in client.stream(...)
    async for token in llm.stream(prompt):
        yield token

@app.post("/chat/stream")
async def chat_stream(request: Request, body: dict):
    async def event_gen():
        try:
            async for token in token_stream(body["prompt"]):
                if await request.is_disconnected():   # stop paying for tokens nobody reads
                    break
                yield f"data: {json.dumps({'delta': token})}\n\n"
            yield "event: done\ndata: {}\n\n"
        except Exception:
            yield f"event: error\ndata: {json.dumps({'message': 'generation failed'})}\n\n"

    return StreamingResponse(
        event_gen(),
        media_type="text/event-stream",
        headers={"Cache-Control": "no-cache", "X-Accel-Buffering": "no"},  # disable nginx buffering
    )
```

Points to mention:
- The SSE format is `data: ...\n\n`; JSON-encode each payload so newlines in tokens don't break framing.
- Streaming improves **perceived latency**: time to first token is maybe 300 ms instead of an 8 s wait for the full answer.
- Handle client disconnects to cancel upstream generation.
- Proxies and load balancers buffer by default; disable buffering and raise idle timeouts.
- Guardrails on streamed output: buffer at sentence level and check before flushing, or check the complete answer and send a correction event.
- The browser uses `EventSource` (GET only) or `fetch` with a stream reader for POST.

**Follow-ups:**
- *SSE vs WebSocket?* → SSE is one-way over plain HTTP, simpler, and auto-reconnects. WebSocket is bidirectional, needed for voice or interruptions.

**Red flags:**
- Returning the full response and calling it streaming.

---

<a id="q8"></a>
## Q8. Coding: Maximum Subarray Sum (Kadane's algorithm).
*Source: interview posts*

> Given an integer array `arr[]`, find the subarray (at least one element) with the maximum sum.
> `[2, 3, -8, 7, -1, 2, 3]` → `11`; `[-2, -4]` → `-2`; `[5, 4, 1, 7, 8]` → `25`.

**What the interviewer is testing:** Moving from brute force to an optimal DP insight, handling edge cases, and explaining clearly.

**Strong answer:**

"Brute force checks all O(n²) subarrays. The better insight: at each index, the best subarray **ending here** either extends the previous best-ending-here or starts fresh at this element. That's Kadane's algorithm. It is O(n) time and O(1) space."

```python
def max_subarray_sum(arr: list[int]) -> int:
    if not arr:
        raise ValueError("array must contain at least one element")
    best = cur = arr[0]            # init with first element -> handles all-negative arrays
    for x in arr[1:]:
        cur = max(x, cur + x)      # extend or restart
        best = max(best, cur)
    return best

assert max_subarray_sum([2, 3, -8, 7, -1, 2, 3]) == 11   # 7 + (-1) + 2 + 3
assert max_subarray_sum([-2, -4]) == -2                  # single largest element
assert max_subarray_sum([5, 4, 1, 7, 8]) == 25           # whole array
```

Trace for `[2, 3, -8, 7, -1, 2, 3]`:

| x | cur | best |
|---|---|---|
| 2 | 2 | 2 |
| 3 | 5 | 5 |
| -8 | -3 | 5 |
| 7 | 7 (restart) | 7 |
| -1 | 6 | 7 |
| 2 | 8 | 8 |
| 3 | 11 | 11 |

Edge cases: all negatives (initializing with `0` would wrongly return 0; initializing with `arr[0]` is correct), a single element, and an empty array (clarify the contract).

**Follow-ups:**
- *Return the indices too?* → Track `start` when you restart, and record `(start, i)` whenever `best` updates.
- *Circular array?* → max(normal Kadane, total − min-subarray), unless all elements are negative.
- *Divide and conquer?* → O(n log n); worth mentioning, but Kadane is better.

**Red flags:**
- Initializing `best = 0`, which fails `[-2, -4]`.
- Not testing the given examples.

---

<a id="q9"></a>
## Q9. Coding: implement Reciprocal Rank Fusion.
*Source: interview posts*

**What the interviewer is testing:** Can you implement hybrid-search fusion and explain why it uses ranks instead of scores?

**Strong answer:**

RRF combines ranked lists without normalizing incompatible scores (BM25 scores are unbounded; cosine similarity is in [-1, 1]).
`score(d) = Σ over lists of 1 / (k + rank_d)`, with k = 60 by convention (from the original paper), and ranks starting at 1.

```python
from collections import defaultdict

def rrf(rankings: list[list[str]], k: int = 60, top_n: int | None = None):
    scores: dict[str, float] = defaultdict(float)
    for ranking in rankings:
        for rank, doc_id in enumerate(ranking, start=1):
            scores[doc_id] += 1.0 / (k + rank)
    fused = sorted(scores.items(), key=lambda kv: kv[1], reverse=True)
    return fused[:top_n] if top_n else fused

bm25  = ["d3", "d1", "d7", "d2"]
dense = ["d1", "d4", "d3", "d9"]
print(rrf([bm25, dense], top_n=5))
# d1 0.03252 (ranks 2,1), d3 0.03227 (ranks 1,3), d4 0.01613, d7 0.01587, d2 0.01562
```

Why k = 60: a larger k flattens the difference between rank 1 and rank 5, so a document that appears in both lists beats one that tops a single list. A smaller k rewards top ranks more. Weighted RRF (`w_i / (k + rank)`) lets you favor BM25 for ID-heavy queries.

Complexity: O(total items + U log U) for U unique docs.

**Follow-ups:**
- *Alternative?* → Score normalization (min-max or z-score) plus a weighted sum; it is sensitive to score distributions per query.
- *After RRF?* → A cross-encoder reranker on the fused top 20–50.

**Red flags:**
- Adding raw BM25 and cosine scores directly.

---

<a id="q10"></a>
## Q10. Coding: implement a chunker with overlap and a cosine top-k search.
*Source: interview posts*

**What the interviewer is testing:** Building RAG primitives from scratch, and edge cases.

**Strong answer:**

```python
import numpy as np

def chunk_text(text: str, chunk_size: int = 200, overlap: int = 40) -> list[str]:
    """Word-based sliding window. Production: token-based + respect sentence/heading boundaries."""
    if overlap >= chunk_size:
        raise ValueError("overlap must be smaller than chunk_size")
    words = text.split()
    step = chunk_size - overlap
    chunks = []
    for start in range(0, len(words), step):
        chunks.append(" ".join(words[start:start + chunk_size]))
        if start + chunk_size >= len(words):      # last window reached the end
            break
    return chunks
# 500 words, size 200, overlap 40 -> windows at 0,160,320 -> 3 chunks (200, 200, 180 words)

def cosine_top_k(query_vec: np.ndarray, doc_matrix: np.ndarray, k: int = 5):
    q = query_vec / np.linalg.norm(query_vec)
    d = doc_matrix / np.linalg.norm(doc_matrix, axis=1, keepdims=True)
    sims = d @ q                                   # (n_docs,)
    k = min(k, len(sims))
    idx = np.argpartition(-sims, k - 1)[:k]        # O(n) selection
    idx = idx[np.argsort(-sims[idx])]              # sort only the k winners
    return list(zip(idx.tolist(), sims[idx].tolist()))
```

Talking points:
- **Pre-normalize** document vectors once at index time; then cosine similarity is just a dot product.
- `argpartition` is O(n) instead of O(n log n) for a full sort.
- Brute force is fine up to about 100k–1M vectors on a CPU (1M × 768 float32 ≈ 3 GB). Beyond that, use ANN (HNSW/IVF).
- Real chunkers count **tokens** (tiktoken or the embedding model's tokenizer), split on semantic boundaries (headings, paragraphs), and store metadata (doc_id, page, offsets) for citations.
- Guard against zero vectors (division by zero) and empty text.

**Follow-ups:**
- *Why overlap?* → So a sentence that spans a boundary is fully present in at least one chunk. 10–20% is typical.
- *Memory reduction?* → float16 or int8 quantization of vectors, or PQ.

**Red flags:**
- An infinite loop when `overlap >= chunk_size`.
- A full sort of millions of scores in the hot path.

---

<a id="q11"></a>
## Q11. HTTP fundamentals for AI engineers: status codes (429, 5xx), idempotency keys, timeouts.
*[Added]*

**What the interviewer is testing:** Robust client/server behavior around LLM APIs and tools.

**Strong answer:**

Status codes you must handle:

| Code | Meaning | Action |
|---|---|---|
| 400 / 422 | Bad request / validation (e.g. context too long) | Don't retry; fix the input (truncate, re-chunk) |
| 401 / 403 | Auth / permission | Don't retry; refresh the token or fail |
| 404 | Not found | Don't retry |
| 408 / timeout | Request timeout | Retry with backoff if idempotent |
| 409 | Conflict | Re-read state, then decide |
| 429 | Rate limited | Retry after `Retry-After`, with backoff; reduce concurrency |
| 500 / 502 / 503 / 504 | Server / gateway errors | Retry with backoff; circuit breaker; failover provider |
| 529 / provider "overloaded" | Capacity | Backoff plus failover |

**Idempotency:** GET, PUT and DELETE are idempotent by definition; POST is not. For side-effecting POSTs (create a ticket, charge a card, send an email), the client sends an `Idempotency-Key` header (a UUID per logical operation), and the server stores the key → result mapping so a retry returns the original result instead of executing twice. This is essential for agents that retry tool calls.

**Timeouts:** set connect and read timeouts explicitly (the httpx default is 5 s, which is often too short for LLM generation; unbounded is worse). Use a total deadline per user request and propagate the remaining budget to downstream calls.

**Other points:** keep-alive and connection reuse (one shared client), gzip, pagination, ETags for caching, and CORS for browser clients.

**Follow-ups:**
- *Streaming timeout?* → Use an idle timeout between chunks rather than a total read timeout.

**Red flags:**
- Retrying every 4xx.
- No idempotency keys on retried side effects.

---

<a id="q12"></a>
## Q12. What is Pydantic's role in GenAI apps?
*[Added]*

**What the interviewer is testing:** Using types as the contract between probabilistic model output and deterministic code.

**Strong answer:**

Pydantic is the **boundary validator** between the LLM and the rest of the system:

1. **API contracts:** FastAPI request and response models.
2. **Structured LLM output:** define the schema once, pass its JSON schema to the model (tool calling / structured outputs), then validate the result.
3. **Tool definitions:** tool argument schemas generated from models (LangChain, OpenAI and Anthropic SDKs all accept JSON schema).
4. **Settings:** `pydantic-settings` for env and config management.
5. **Validation-repair loop:** on `ValidationError`, send the error back to the model once ("field `amount` must be > 0"), then fail safely.

```python
from pydantic import BaseModel, Field, field_validator
from typing import Literal

class RefundDecision(BaseModel):
    action: Literal["approve", "reject", "escalate"]
    amount_inr: float = Field(ge=0, le=5000)
    reason: str = Field(min_length=10)

    @field_validator("reason")
    @classmethod
    def no_pii(cls, v: str) -> str:
        if "@" in v:
            raise ValueError("reason must not contain emails")
        return v

decision = RefundDecision.model_validate_json(llm_output)   # raises on invalid output
```

Key point: `Literal`, `Field` bounds and validators encode **business rules in code**. The model can't "decide" a ₹50,000 refund because validation rejects it before any action runs. Libraries like `instructor` wrap this validate-and-retry pattern.

**Follow-ups:**
- *Pydantic v2 vs v1?* → The v2 core is written in Rust (much faster); the API changed (`model_validate`, `field_validator`).

**Red flags:**
- Using `json.loads` and trusting the dict.

---

## Rapid-fire recap

- Async is for I/O-bound concurrency; CPU-bound work goes to processes or workers.
- Blocking calls inside `async def` freeze the event loop.
- FastAPI = ASGI + Pydantic + OpenAPI + dependency injection.
- Build DB URLs with `URL.create`; load secrets from a vault or managed identity.
- Pools are per process: pods × workers × (pool_size + max_overflow) must fit the DB limit.
- Never hold a DB connection while waiting for an LLM.
- Bound LLM concurrency with a semaphore; retry 429/5xx with jittered exponential backoff.
- Retried side effects need idempotency keys.
- SSE: `data: ...\n\n`, disable proxy buffering, stop on client disconnect.
- Kadane: `cur = max(x, cur + x)`; initialize with `arr[0]`, not 0. O(n) time, O(1) space.
- RRF: `Σ 1/(60 + rank)`; ranks avoid score-scale mismatch.
- Pydantic enforces business rules on model output before any action runs.
