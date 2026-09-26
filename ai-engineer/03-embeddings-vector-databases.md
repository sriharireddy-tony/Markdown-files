# 03 — Embeddings & Vector Databases

**What interviewers probe here:** whether you can pick an embedding model and a vector store *for a reason*, explain how similarity search really works (ANN indexes, not magic), and handle the operational side: local vs production, version mismatches, re-embedding migrations and cost.

[← Back to index](README.md)

1. [What an embedding is and why we need it](#q1)
2. [How text becomes a vector](#q2)
3. [Choosing an embedding model and dimension](#q3)
4. [How similarity search works](#q4)
5. [Choosing a vector DB](#q5)
6. [Local vs production](#q6)
7. [Build a vector store from scratch](#q7)
8. [Detecting embedding version mismatch](#q8)
9. [Zero-downtime embedding migration](#q9)
10. [Pre-filtering vs post-filtering](#q10)
11. [Domain jargon: fine-tune embeddings?](#q11)
12. [Exploding vector storage cost](#q12)

---

<a id="q1"></a>
## Q1. What is an embedding and why do we need it?
*Source: interview posts*

**What the interviewer is testing:** A crisp definition plus the "why" in the context of a real system.

**Strong answer:**

"An embedding is a dense, fixed-length vector of floats (e.g. 768 or 1536 dimensions) that represents a piece of content so that **semantically similar items are close together** in that space.

We need it because computers can't compare meaning directly. Keyword matching fails when the user writes 'how do I get my money back' and the policy says 'refund process'. Embeddings turn meaning comparison into cheap geometry (cosine similarity), which enables:
- Semantic search and RAG retrieval
- Deduplication and clustering (e.g. grouping support tickets)
- Recommendations and 'similar items'
- Classification features, anomaly detection, semantic caching

A concrete example from an HR assistant: 'Can I carry forward unused leave?' matched a chunk titled 'Leave encashment and rollover rules' at 0.82 cosine similarity with zero keyword overlap on 'carry forward'. BM25 alone missed it."

**Follow-ups:**
- *Limitations?* → Weak on exact identifiers, numbers and negation. It's lossy, and its behavior depends on training data. Pair it with keyword search.

**Red flags:**
- "Embeddings store the text in the vector database."

---

<a id="q2"></a>
## Q2. How does text get converted into a vector?
*Source: interview posts*

**What the interviewer is testing:** The actual mechanics inside an embedding model.

**Strong answer:**

1. **Tokenize** the text into sub-word IDs, truncated at the model's max length (often 512 tokens, some 8k+).
2. **Token embedding lookup** plus position information.
3. **Transformer encoder layers** produce a contextual vector per token. 'Bank' near 'river' gets a different vector from 'bank' near 'loan'.
4. **Pooling** reduces the per-token vectors to one: mean pooling, the [CLS] token, or last-token pooling for decoder-based embedders.
5. **Normalize** (L2) so cosine similarity equals the dot product.

The model was trained **contrastively** so the pooled vectors of relevant query–passage pairs are close.

"Practical gotchas I check:
- **Truncation:** text beyond the max tokens is silently dropped, so chunk size must fit the embedder.
- **Instruction prefixes:** some models (E5, BGE and others) expect prefixes like `query:` / `passage:`, or a task instruction. Omitting them can cost several points of recall.
- **Use the same model and preprocessing** for queries and documents."

```python
from sentence_transformers import SentenceTransformer
m = SentenceTransformer("BAAI/bge-small-en-v1.5")
v = m.encode(["refund process for annual plans"], normalize_embeddings=True)
print(v.shape)   # (1, 384)
```

**Follow-ups:**
- *Asymmetric vs symmetric search?* → A short query vs a long passage (asymmetric) needs models trained for it, or query/passage prefixes.

**Red flags:**
- Not knowing about truncation limits.

---

<a id="q3"></a>
## Q3. How do you choose an embedding model? How does dimension affect quality, storage and latency, and which dimension would you pick?
*Source: interview posts*

**What the interviewer is testing:** Evidence-based selection and cost awareness.

**Strong answer:**

"I choose on **my data**, not on leaderboard position. The factors:

| Factor | Question |
|---|---|
| Retrieval quality | Recall@10 / nDCG on *my* labeled query set (MTEB is only a shortlist) |
| Language/domain | Multilingual? Code? Legal or medical jargon? |
| Max input length | Does it fit my chunk size? |
| Dimension | Storage, RAM and search speed |
| Latency/throughput | Can I embed 10M chunks in hours, and queries in <30 ms? |
| Hosting | API (data leaves my network) vs self-hosted (compliance) |
| Cost | $ per million tokens, or GPU cost |
| Matryoshka support | Can I truncate dimensions without re-embedding? |

**Dimension trade-off:** storage = N × d × 4 bytes (float32). For 10M chunks:
- 384 dims → about 15 GB
- 1024 dims → about 41 GB
- 3072 dims → about 123 GB (before index overhead, which is often +30–100% for HNSW)

Higher dimensions capture more nuance, but returns diminish. Beyond about 768–1024, quality gains are usually small relative to 2–3x memory and slower distance computation.

**What I'd pick:** start with a strong mid-size model at 768–1024 dims. If it supports **Matryoshka** representation learning (e.g. OpenAI text-embedding-3, Nomic, some BGE/Jina variants), test truncating to 256–512 dims. Often you keep about 95–98% of the quality at a third of the storage. On one 8M-chunk corpus we went 1536 → 512 and lost 1.2 points of recall@10 while cutting the vector DB bill by about 60%."

**Follow-ups:**
- *Why not just use MTEB's top model?* → The benchmark domain differs from mine, top models are often large and slow, and some overfit the benchmark.
- *Multilingual?* → Use a multilingual model (e.g. BGE-M3, multilingual-e5) and test cross-lingual queries explicitly.

**Red flags:**
- "Bigger dimension is always better."
- Choosing without an eval set.

---

<a id="q4"></a>
## Q4. How does similarity search work (cosine/dot/L2, ANN: HNSW, IVF, PQ)?
*Source: interview posts*

**What the interviewer is testing:** Whether you know the index trade-offs you're tuning.

**Strong answer:**

"**Distance metrics:**
- Cosine: angle only, which ignores magnitude.
- Dot product: equals cosine when vectors are normalized. Fastest.
- L2 (Euclidean): gives the same ranking as cosine for normalized vectors.

Always use the metric the model was trained with.

**Exact search** (brute force) is O(N·d) per query. That's fine up to around 100k–1M vectors on a GPU/CPU with batching, but too slow at 100M. So we use **Approximate Nearest Neighbor** indexes that trade a little recall for big speed gains:

| Index | Idea | Pros | Cons | Key knobs |
|---|---|---|---|---|
| HNSW | Multi-layer proximity graph; greedy walk from coarse to fine layers | Excellent recall/latency, incremental inserts | High RAM, slow builds, deletes are awkward | M, ef_construction, ef_search |
| IVF | k-means into nlist clusters; search the nprobe nearest clusters | Less memory, fast builds | Recall depends on nprobe; needs training | nlist, nprobe |
| PQ / SQ | Compress vectors (product/scalar quantization) | 4–32x less memory | Recall loss; needs rescoring | code size, bits |
| DiskANN | Graph on SSD | Billion scale on less RAM | More engineering | — |

**Tuning example:** HNSW with M=16, ef_construction=200. Raising ef_search from 40 to 128 took recall@10 (vs exact) from 0.92 to 0.99, and P95 went from 4 ms to 11 ms. We chose 96, the knee of the curve.

I always measure **ANN recall against brute force** on a sample, so I know how much the index itself is losing, separately from embedding quality."

**Follow-ups:**
- *Why is HNSW bad with heavy deletes?* → Deleted nodes are tombstoned and the graph degrades, so it needs periodic rebuild or compaction.

**Red flags:**
- Thinking the vector DB does exact search at scale.
- Not knowing the recall/latency knobs.

---

<a id="q5"></a>
## Q5. How do you choose a vector DB? Compare FAISS, Pinecone, Qdrant, Chroma, Weaviate and pgvector.
*Source: interview posts*

**What the interviewer is testing:** Product judgment. There's no single right answer, but you need clear criteria.

**Strong answer:**

| Option | What it is | Strengths | Watch-outs | Typical fit |
|---|---|---|---|---|
| FAISS | A library (Meta), not a database | Fastest raw ANN, GPU support, many index types | No persistence/API/filters/replication out of the box; you build the service | Research, batch jobs, embedded in your own service |
| Chroma | Lightweight embedded/dev DB | Zero setup, Python-native | Limited scale and ops features historically | Prototyping, local dev, small apps |
| pgvector | Postgres extension (HNSW/IVFFlat) | One DB for relational + vectors, SQL joins, transactions, existing backups/RBAC | Tuning at 10M+ vectors, memory pressure, less advanced hybrid search | Teams already on Postgres, <10–50M vectors, strong filtering needs |
| Qdrant | Open-source vector DB (Rust); self-host or cloud | Fast, rich payload filtering, quantization, sparse+dense hybrid, multi-tenancy | You operate it (unless using cloud) | Production self-hosted, filter-heavy RAG |
| Weaviate | Open-source vector DB; self-host or cloud | Built-in hybrid BM25+vector, modules, multi-tenancy | Heavier ops, resource use | Hybrid-search-first apps |
| Pinecone | Fully managed SaaS | No ops, serverless scaling, namespaces | Vendor lock-in, cost at scale, data residency limits | Teams wanting zero ops, fast time to market |

(Milvus is also worth mentioning for very large, distributed deployments.)

**My decision criteria, in order:** data residency and compliance → scale (vectors, QPS) → filtering and multi-tenancy needs → hybrid search → ops capacity of the team → cost → ecosystem.

**Example decision:** 3M chunks, strict metadata filters by tenant and permission, data must stay in our VPC, and a team already running Postgres. I chose **pgvector** with HNSW, and SQL joins solved the permissions problem elegantly. For a different client with 200M vectors and heavy hybrid search, I'd choose Qdrant or Milvus self-hosted, or a managed option if ops capacity is thin."

**Follow-ups:**
- *When would you migrate off pgvector?* → When index size exceeds comfortable RAM, P95 degrades under concurrent writes, or advanced hybrid/quantization features are needed.

**Red flags:**
- "Pinecone is the best."
- Calling FAISS a database.

---

<a id="q6"></a>
## Q6. What would you use for local development vs production, and why (scalability, availability, performance, cost, security, backup, maintenance)?
*Source: interview posts*

**What the interviewer is testing:** Ops maturity and environment parity.

**Strong answer:**

"**Local:** Chroma, or Qdrant/pgvector in **Docker**. Fast iteration, no cost, runs offline. I actually prefer running the *same engine* locally in Docker as in production (e.g. Qdrant or pgvector), to avoid 'works on Chroma, breaks on Qdrant' surprises in filtering syntax and scoring.

**Production:** a managed or clustered deployment of that engine. Here's how I'd reason through each criterion:

| Criterion | Local | Production |
|---|---|---|
| Scalability | Single node, sample data | Sharding/replicas; capacity plan for 3x growth |
| Availability | Irrelevant | ≥2 replicas across AZs, health checks, 99.9% SLO |
| Performance | Good enough | Tuned index params, quantization, P95 < 50 ms target, load-tested |
| Cost | Free | Right-sized RAM (the index must fit), reserved capacity or serverless |
| Security | Local only | VPC/private endpoints, TLS, API keys/IAM, encryption at rest, per-tenant isolation |
| Backup & recovery | None | Scheduled snapshots, tested restore; can also rebuild from the source of truth |
| Maintenance | None | Upgrades, reindex/compaction, monitoring (latency, memory, recall canaries) |

A key point I always make: **the vector DB is not the source of truth.** Raw documents and chunk metadata live in object storage or a primary DB, with versioned embeddings. The vector index is a derived artifact I can rebuild. That simplifies backup strategy and embedding-model migrations."

**Follow-ups:**
- *Seed data locally?* → A small representative corpus plus the eval set, with fixture scripts in the repo.

**Red flags:**
- Using an in-memory FAISS index in production with no persistence plan.

---

<a id="q7"></a>
## Q7. Build a vector store from scratch with no Chroma or Pinecone. What do you implement?
*Source: interview posts (30-day build roadmap)*

**What the interviewer is testing:** Whether you understand what vector DBs do under the hood.

**Strong answer:**

"Minimum viable features: add vectors with IDs and metadata, normalize, top-k cosine search with metadata filtering, delete, and persist. Brute force with NumPy is exact and fine up to around a few hundred thousand vectors."

```python
import numpy as np, json, pathlib

class TinyVectorStore:
    def __init__(self, dim: int):
        self.dim = dim
        self.vecs = np.empty((0, dim), dtype=np.float32)
        self.ids: list[str] = []
        self.meta: list[dict] = []

    @staticmethod
    def _norm(x: np.ndarray) -> np.ndarray:
        return x / (np.linalg.norm(x, axis=-1, keepdims=True) + 1e-12)

    def add(self, ids, vectors, metadatas):
        v = self._norm(np.asarray(vectors, dtype=np.float32))
        assert v.shape[1] == self.dim, "dimension mismatch"
        self.vecs = np.vstack([self.vecs, v])
        self.ids += list(ids); self.meta += list(metadatas)

    def delete(self, doc_id: str):
        keep = [i for i, m in enumerate(self.meta) if m.get("doc_id") != doc_id]
        self.vecs = self.vecs[keep]
        self.ids = [self.ids[i] for i in keep]; self.meta = [self.meta[i] for i in keep]

    def search(self, query_vec, k=5, where: dict | None = None):
        q = self._norm(np.asarray(query_vec, dtype=np.float32))
        mask = np.ones(len(self.ids), dtype=bool)
        if where:  # pre-filter: exact match on metadata fields
            mask = np.array([all(m.get(f) == v for f, v in where.items()) for m in self.meta])
        idx = np.where(mask)[0]
        if idx.size == 0:
            return []
        scores = self.vecs[idx] @ q                     # cosine, since both are normalized
        k = min(k, idx.size)
        top = idx[np.argpartition(-scores, k - 1)[:k]]  # O(n) selection instead of a full sort
        top = sorted(top, key=lambda i: -float(self.vecs[i] @ q))
        return [(self.ids[i], float(self.vecs[i] @ q), self.meta[i]) for i in top]

    def save(self, path: str):
        p = pathlib.Path(path); p.mkdir(parents=True, exist_ok=True)
        np.save(p / "vecs.npy", self.vecs)
        (p / "meta.json").write_text(json.dumps({"dim": self.dim, "ids": self.ids, "meta": self.meta}))
```

"Then I'd explain what a real DB adds on top: an **ANN index** (HNSW/IVF) for sub-linear search, **write-ahead logs** and crash safety, concurrent reads/writes, **payload indexes** for fast filtering, sharding and replication, quantization, hybrid sparse search, and access control. Building this makes it obvious why deletes, filters and updates are the hard parts, not the dot product."

**Follow-ups:**
- *How would you scale this?* → Add HNSW (e.g. `hnswlib`), memory-map the vector file, batch the queries, and shard by tenant.
- *Complexity?* → Search is O(N·d). argpartition gives O(N) selection.

**Red flags:**
- Forgetting normalization or metadata filtering.
- Sorting the whole array on every query without mentioning it.

---

<a id="q8"></a>
## Q8. How do you detect an embedding model version mismatch?
*Source: interview posts*

**What the interviewer is testing:** Operational rigor. It's a silent, common failure.

**Strong answer:**

"A mismatch happens when queries are embedded with a different model or version (or different preprocessing, prefixes or normalization) than the indexed documents. Vectors from different models live in **incompatible spaces**. The search still returns results, just bad ones, so nothing errors.

**Prevention:**
- Store `embedding_model`, `model_version`, `dim`, `preprocess_version` as metadata on every vector **and** on the collection. Name collections by model, e.g. `docs__bge-m3__v2`.
- The query service reads the collection's model config and refuses to query with a different one (fail fast).
- Pin model versions. API providers can update models, so use dated model IDs.

**Detection:**
- A dimension check (catches only the crudest cases).
- **Canary queries:** a fixed set of 50 queries with known relevant doc IDs, run every hour. If recall@5 drops from 0.95 to 0.4, alert.
- **Score distribution monitoring:** average top-1 similarity falling from about 0.78 to 0.45 overnight is a classic mismatch signature.
- **Self-retrieval test:** re-embed a sample of stored chunk texts with the current query-side model and check that cosine similarity to their stored vectors is ≈1.0. Anything well below means the spaces differ.

Real incident: a library upgrade changed the default pooling for our self-hosted embedder. Top-1 similarity dropped by 0.2, and the canary caught it within an hour."

**Follow-ups:**
- *Can you mix models in one index?* → No. Re-embed everything, or keep separate collections and fuse the results.

**Red flags:**
- Not knowing that a mismatch fails silently.

---

<a id="q9"></a>
## Q9. How do you migrate a live index to a new embedding model with zero downtime?
*[Added]*

**What the interviewer is testing:** Blue-green thinking for data systems.

**Strong answer:**

1. **Evaluate first:** the new model must beat the old one on the labeled retrieval set by a meaningful margin to justify re-embedding cost.
2. **Build the new index in parallel** (collection `v2`) from the source of truth, with backfill jobs batched and rate-limited.
3. **Dual-write:** new and updated docs go to both `v1` and `v2` during the backfill, so `v2` doesn't fall behind.
4. **Verify parity:** document counts, canary queries, recall on the eval set.
5. **Shadow traffic:** run real queries against `v2`, compare the overlap of results and judge scores.
6. **Cut over** with an alias/config switch (the query embedder and the collection switch *together*), gradually through a percentage rollout.
7. **Keep `v1`** for a rollback window (e.g. 7 days), then delete it.

"Cost estimate example: 20M chunks × 300 tokens = 6B tokens. At $0.02 per M tokens that's about $120 via API. Self-hosted, it's about 6–10 GPU-hours. Cheap relative to the engineering risk, but plan the write throughput of the vector DB."

**Red flags:**
- Re-embedding in place over the live collection.

---

<a id="q10"></a>
## Q10. Metadata filtering with ANN: what goes wrong with pre-filtering vs post-filtering?
*[Added]*

**What the interviewer is testing:** A subtle correctness/performance trap.

**Strong answer:**

- **Post-filtering:** get the ANN top-k, then drop results that fail the filter. With a selective filter (e.g. tenant = 1% of data), top-10 → filter → 0–1 results. **You silently lose recall.**
- **Naive pre-filtering:** filter first, then brute-force the subset. Correct, but slow if the subset is large, and it breaks HNSW's graph traversal if done badly.
- **Filtered ANN** (what Qdrant, Weaviate, Pinecone and pgvector with iterative scans do): apply the filter *during* graph traversal, and fall back to brute force when the filtered set is small.

"My approach: high-selectivity, high-cardinality keys like tenant get **separate collections, namespaces or partitions**. Medium filters use payload indexes and filtered search. Always test recall *with* filters, because unfiltered benchmarks hide this."

**Red flags:**
- Assuming filters are free.

---

<a id="q11"></a>
## Q11. Retrieval is poor on domain jargon. Do you fine-tune the embedding model?
*[Added]*

**What the interviewer is testing:** Choosing the cheapest effective fix.

**Strong answer:**

"Not first. Cheaper fixes, in order:
1. **Hybrid search.** BM25 catches exact jargon, acronyms and codes.
2. **Query expansion** with a glossary or an LLM rewrite ('PTO' → 'paid time off').
3. **A reranker** (a cross-encoder handles nuance better).
4. **Try a different embedding model** (domain-specific or larger).
5. **Then fine-tune:** generate about 5–50k (query, positive) pairs, using an LLM to write questions for chunks, mine **hard negatives** with the current retriever, and train with MultipleNegativesRankingLoss. Evaluate on a held-out, human-labeled set, and re-embed the corpus afterwards.

Typical outcome: +8–15 points recall@10 on specialized corpora. The costs are ongoing ownership, re-embedding and a hosting requirement."

**Red flags:**
- Jumping straight to fine-tuning with no baseline.

---

<a id="q12"></a>
## Q12. Vector storage cost is exploding. What are your options?
*[Added]*

**What the interviewer is testing:** Cost engineering on the retrieval side.

**Strong answer:**

- **Reduce N:** dedupe near-duplicate chunks (often 10–30% in enterprise corpora), drop boilerplate, and archive stale docs.
- **Reduce d:** Matryoshka truncation (1536 → 512).
- **Quantize:** scalar int8 (4x smaller, ~1% recall loss), binary (32x smaller) with **rescoring** on full-precision vectors for the top candidates.
- **Tiered storage:** keep the hot index in RAM, put cold tenants on disk (DiskANN or memory-mapped storage).
- **Right-size the index:** lower HNSW M, or IVF-PQ for huge cold collections.
- **Rethink the chunking:** parent-child retrieval (embed small, return big) instead of heavily overlapping chunks.

"Example: 40M × 1536 float32 ≈ 245 GB RAM. With 512-dim int8 plus dedupe it came to about 16 GB, recall@10 dropped 1.8 points, and cost fell by more than 85%."

**Red flags:**
- Only saying "buy a bigger instance".

---

## Rapid-fire recap
- Embeddings turn meaning comparison into geometry. They're weak on exact IDs, numbers and negation.
- The pipeline is tokenize → contextual encoder → pooling → normalize. Watch truncation and query/passage prefixes.
- Choose embedding models on your own labeled eval set. MTEB is a shortlist.
- Storage = N × d × 4 bytes plus index overhead. Matryoshka and quantization cut it sharply.
- HNSW gives fast, high recall with high RAM. IVF/PQ give lower memory with lower recall. Tune ef_search/nprobe.
- FAISS is a library, pgvector is Postgres, and Qdrant, Weaviate and Pinecone are databases. Choose by compliance, scale, filters and ops.
- The vector DB is a derived index. Documents plus metadata are the source of truth.
- Model mismatch fails silently. Store model metadata, fail fast, run canary queries.
- Migrate embeddings blue-green: parallel index, dual-write, shadow, alias switch.
- Post-filtering loses recall on selective filters. Partition by tenant.
- Fix jargon with hybrid search, expansion and a reranker before fine-tuning embeddings.
