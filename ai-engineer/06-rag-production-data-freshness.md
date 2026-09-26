# 06 — RAG in Production & Data Freshness

What interviewers probe here: whether you treat the index as a living production dataset, with versioning, deletes, permissions, tenancy, failure recovery and latency budgets, and not as a one-time "embed the PDFs" script. Most real RAG incidents come from this layer, not from the model.

[← Back to index](README.md)

1. [Real-time KB updates without stale embeddings](#q1-the-knowledge-base-updates-in-real-time-how-do-you-avoid-stale-embeddings-and-stop-stale-documents-being-returned)
2. [Section-level edits and removals](#q2-document-versioning-how-do-you-handle-section-level-edits-and-complete-section-removal-while-keeping-the-index-consistent)
3. [Failed re-indexing](#q3-how-would-you-safely-handle-a-failed-re-indexing)
4. [Ingestion for 50M documents](#q4-design-ingestion-for-50m-documents)
5. [Changed document permissions](#q5-how-do-you-handle-changed-document-permissions)
6. [Multi-tenant RAG isolation](#q6-design-multi-tenant-rag-where-tenants-must-never-see-each-others-data)
7. [Where RAG latency comes from](#q7-where-does-most-rag-latency-come-from-and-how-do-you-reduce-it)
8. [Near-duplicate and conflicting documents](#q8-near-duplicate-and-conflicting-documents-are-polluting-retrieval-what-do-you-do)
9. [The production layer around RAG](#q9-what-does-the-production-layer-around-rag-look-like-caching-re-indexing-jobs-costlatency-tracking-pii)

---

## Q1. The knowledge base updates in real time. How do you avoid stale embeddings and stop stale documents being returned?
*Source: interview posts*

**What the interviewer is testing:** Whether you design the index as a change-data-capture (CDC) pipeline with deletes and versioning, not as a nightly batch job.

**Strong answer:**

"First I'd ask how fresh the data needs to be. 'Real time' usually means seconds for prices or inventory and minutes for policies. Staleness has two sources: **new content not yet indexed**, and **old content never removed**. The second is the dangerous one, because the model confidently cites a price that no longer exists.

My design:

```
Source system ──CDC/webhook──> Queue (Kafka/SQS) ──> Ingest workers
                                                     │ parse → chunk → hash
                                                     │ embed only changed chunks
                                                     ▼
                                  Vector DB (upsert by chunk_id, delete by doc_id)
```

- **Stable IDs**: `chunk_id = doc_id + section_path + chunk_index`, plus a `content_hash`. If the hash is unchanged, skip re-embedding. That typically cuts embedding cost by 80–90% on edits.
- **Delete before insert** per document, or tag every chunk with a `doc_version` and filter on `is_current = true` at query time.
- **Tombstones** for deleted source docs, so the delete event is never lost.
- **Freshness metadata** (`updated_at`, `valid_until`) on every chunk. For volatile facts I filter or boost by recency.
- **Don't embed volatile facts at all.** Prices, stock levels and order status should come from a **live tool/API call**, not the vector index. RAG serves the policy text; the tool serves the number.

Trade-off: streaming ingestion costs more to operate than a nightly batch. I use streaming only for sources with a freshness SLA and keep batch for everything else.

Example: in a multi-business voice agent where prices changed weekly, we indexed the policy documents but fetched prices from the business's catalog API at call time. Stale-price complaints dropped to zero, and re-indexing stopped being on the critical path."

**Follow-ups:**
- *How do you measure staleness?* → Track index lag (source `updated_at` minus indexed `updated_at`) as a metric with an alert, e.g. P95 under 5 minutes.
- *What about cached answers?* → Invalidate the semantic cache by `doc_id` tags when a source doc changes, or keep TTLs short for volatile domains.
- *Upsert or delete+insert?* → Delete+insert per document if chunk counts can change, since upsert leaves orphan chunks when a doc shrinks.

**Red flags:**
- "Just re-index everything every night": ignores deletes, cost and the freshness SLA.
- Never mentions deletions or orphan chunks.
- Puts rapidly changing numeric facts in the vector DB.

---

## Q2. Document versioning: how do you handle section-level edits and complete section removal while keeping the index consistent?
*Source: interview posts*

**What the interviewer is testing:** Structural chunking, stable identifiers, and atomic consistency between the document and its chunks.

**Strong answer:**

"The key is to chunk by **document structure** and give every section a **stable identity**. Otherwise one edited paragraph shifts every chunk boundary and you end up re-embedding the whole doc.

1. **Parse to a section tree**: headings → `section_path` like `HR-Policy/4.2/Leave-Carryover`.
2. **Chunk within sections**, never across them. Chunk ID = `doc_id:section_path:idx`.
3. **Diff on re-ingestion** against the previous version's manifest (a list of `section_path → content_hash`):

| Change | Action |
|---|---|
| Section text modified | Re-chunk and re-embed that section only; delete its old chunks |
| Section removed | Delete all chunks with that `section_path` |
| Section added | Embed and insert |
| Section renamed/moved | Treat as delete + add (or match by content hash to reuse embeddings) |
| Unchanged | Skip |

4. **Atomicity**: write new chunks with `doc_version = v2`, then flip a `current_version` pointer for the doc, then garbage-collect v1. Readers filter by current version, so they never see half v1 and half v2.
5. **Keep a manifest store** (Postgres) as the source of truth for what should be in the index, and reconcile the vector DB against it nightly to catch drift.

Trade-off: version filtering adds a metadata filter to every query and more storage until GC runs, but it buys consistency and instant rollback.

Example: for a 3,000-page policy corpus, section-level diffing meant a typical edit re-embedded 5–20 chunks instead of about 400 per document. Reconciliation once caught about 1,200 orphan chunks left by a crashed worker."

**Follow-ups:**
- *Do users ever need old versions?* → For legal or audit use, keep versions queryable with `effective_date` filters ("what was the policy in March?").
- *Chunk-level vs doc-level hash?* → Both: the doc hash to skip entirely, section hashes to diff.
- *How do you handle cross-references to a removed section?* → Flag dangling references during parsing and alert the content owner.

**Red flags:**
- Fixed-size chunking across the whole doc, so any edit reshuffles every chunk.
- No plan for removed content.
- Non-atomic updates where queries see a mix of versions.

---

## Q3. How would you safely handle a failed re-indexing?
*Source: interview posts*

**What the interviewer is testing:** Blue-green thinking, idempotent jobs and validation gates for data pipelines.

**Strong answer:**

"A re-index should never touch the live index in place. I treat it like a deployment:

```
live alias ──> index_v12 (serving)
               index_v13 (building) ── validate ── swap alias ── keep v12 for rollback
```

- **Blue-green indexes with an alias.** Build `v13` fully in the background. Qdrant, Elasticsearch/OpenSearch and Pinecone namespaces all support alias-style swaps; in pgvector, use a new table plus a view swap.
- **Idempotent, checkpointed workers.** Each doc batch is a job keyed by `doc_id + version`. A crash resumes from the last checkpoint, and dead-letter queues hold poison documents instead of failing the whole run.
- **Validation gates before the swap:**
  - Doc and chunk count within ±2% of expectation
  - Embedding dimension and model version correct
  - No null or zero vectors
  - Golden retrieval set: Recall@10 must not drop more than 1–2 points versus v12
- **Swap atomically, keep v12 for 24–72 hours**, then roll back by flipping the alias.
- For incremental updates, a failure means the doc stays at its old version (it's still consistent), and I alert on index lag.

Trade-off: blue-green doubles storage during the build. For a very large index I re-index per shard or per tenant to limit the extra footprint.

Example: we once re-indexed with a tokenizer bug that silently truncated chunks to 128 tokens. The golden-set gate showed Recall@10 falling from 0.91 to 0.74, the swap was blocked, and users never saw it."

**Follow-ups:**
- *Partial failure: 3% of docs failed?* → Swap if under threshold and the failures are in the DLQ with an owner; block if they cluster in one high-traffic source.
- *How do you know the new index is better, not just valid?* → Run an offline eval on the golden set, then shadow traffic, then a small percentage canary.
- *Cost of re-embedding everything?* → Cache embeddings by content hash; only a model change forces a full re-embed.

**Red flags:**
- Dropping and rebuilding the live collection.
- "We retry until it works", with no validation.
- No rollback path.

---

## Q4. Design ingestion for 50M documents.
*Source: interview posts*

**What the interviewer is testing:** Back-of-envelope sizing, distributed pipeline design, cost control and failure handling at scale.

**Strong answer:**

"First I'd clarify: document types (PDF, HTML, scans?), average size, update rate, freshness SLA, and whether there are permissions or multiple tenants. Assume a 50M mixed corpus, 10 chunks per doc on average, so **500M chunks**.

**Sizing:**
- 500M × 1024-dim float32 = ~2 TB of raw vectors. With int8/binary quantization or 512-dim Matryoshka embeddings that becomes ~250–500 GB, so it needs a distributed vector DB (Milvus, Qdrant cluster, Vespa, OpenSearch) or sharded pgvector with care.
- Embedding throughput: at ~300 tokens/chunk, that's 150B tokens. A self-hosted embedding model on GPUs at ~20k chunks/s/cluster takes about 7 hours; an API at scale costs roughly tens of thousands of dollars. That drives a self-host vs API decision, plus batching.

**Pipeline:**
```
Object store (raw) → Discovery/queue → Parse workers (CPU; OCR on GPU for scans)
  → Clean/dedupe (MinHash) → Chunk → Embedding workers (GPU, batched)
  → Bulk loader → Vector DB + BM25 index + metadata DB (manifest)
```
- **Queue-based and horizontally scalable** (Kafka/SQS + Kubernetes jobs, or Spark/Ray for the initial backfill).
- **Separate stages** with persisted intermediate outputs (parsed text in Parquet), so a chunking change doesn't force a re-parse.
- **Dedupe first.** Large corpora often have 20–40% near-duplicates, and removing them saves embedding cost and improves retrieval.
- **Idempotency + DLQ** per document; checkpointing per batch.
- **Bulk-load** into new collections and build the HNSW index after load, not during.
- **Shard** by tenant or source, with metadata stored for filters (ACLs, dates, doc type).
- **Backfill vs incremental**: one big batch job for the backfill, a CDC stream for the steady state.

**Observability:** docs/sec per stage, failure rate by file type, index lag, cost per 1M docs.

Trade-off: the batch-first backfill is cheap and fast; the streaming path is more complex but needed for freshness. Picking 512 vs 1024 dims is a storage vs recall trade-off I'd settle with an eval on a sample."

**Follow-ups:**
- *What's the bottleneck?* → Usually parsing/OCR and embedding throughput, not the vector DB write.
- *How do you validate quality at this scale?* → Sample-based parse QA per file type, plus golden-set retrieval evals per shard.
- *Would you embed everything?* → No. Tier it: hot docs get embedded; cold archives are BM25-only until accessed.

**Red flags:**
- No numbers: no storage, token or time estimates.
- A single-process script "with multiprocessing".
- Ignoring dedupe, parsing failures and file-type diversity.

---

## Q5. How do you handle changed document permissions?
*Source: interview posts*

**What the interviewer is testing:** Whether you understand that authorization must be enforced at retrieval time and permission changes must propagate fast.

**Strong answer:**

"The rule is: **the LLM must never see a chunk the user isn't allowed to read.** Filtering after generation is too late.

Design:
- Store ACL metadata on every chunk: `allowed_groups`, `allowed_users`, `tenant_id`, `classification`.
- At query time, resolve the user's groups from the identity provider (cached for ~5 minutes) and apply a **pre-filter** in the vector search: `allowed_groups ∈ user_groups`.
- **Permission changes are a separate, fast event stream.** When a doc's ACL changes in SharePoint or Drive, I update only the metadata on its chunks (a payload update, no re-embedding). Target propagation: under a minute for revocations.
- **Revocations are more urgent than grants.** On a revocation event I can also add the doc to a short-lived deny-list checked at query time, which covers the window until the metadata update lands.
- **Also invalidate caches.** Semantic and answer caches must be keyed by permission scope, or a cached answer built from a now-restricted doc leaks.
- **Defense in depth:** a post-retrieval check against the source system's ACL API for highly sensitive classes, and an audit log of which chunks served which user.

Trade-off: live ACL checks against the source on every query are the most accurate but add latency (50–200 ms) and hit rate limits. Metadata filtering with event sync is fast but eventually consistent. I use a hybrid: metadata filters for everything, plus live checks only for 'confidential' documents.

Example: an HR doc was moved from 'All employees' to 'HR only' during a reorg. Our sync propagated it in about 40 seconds, and the deny-list covered the gap. Without the cache-key fix, a cached answer would have kept serving it for the full 24-hour TTL."

**Follow-ups:**
- *Groups with 100k members, or nested groups?* → Store group IDs on chunks, never user lists, and expand nested groups on the user side.
- *How do you test it?* → Permission regression tests: a user from group X queries for doc Y and must get zero chunks back.
- *Doc-level vs chunk-level ACLs?* → Usually inherited from the doc; section-level only if the source supports it.

**Red flags:**
- "The prompt tells the LLM not to reveal restricted info."
- Re-embedding documents on permission change.
- Forgetting caches.

---

## Q6. Design multi-tenant RAG where tenants must never see each other's data.
*Source: interview posts*

**What the interviewer is testing:** Isolation models, blast radius, and enforcement that doesn't depend on developer discipline.

**Strong answer:**

"I'd pick an isolation level based on tenant count, size and compliance needs:

| Model | Isolation | Cost/ops | Fits |
|---|---|---|---|
| Shared collection + `tenant_id` filter | Logical | Cheapest | Thousands of small tenants |
| Collection/namespace per tenant | Stronger | Moderate | Hundreds of mid tenants |
| Dedicated cluster/DB per tenant | Physical | Expensive | Regulated/enterprise, data residency |

Often it's a hybrid: shared for SMB tenants, dedicated for enterprise tenants.

Enforcement, whatever the model:
- **`tenant_id` comes from the authenticated token**, never from the request body or the LLM.
- **A data-access layer injects the tenant filter**. Application code cannot call the vector DB without it; direct DB access is blocked. Some DBs support this natively (e.g. Weaviate multi-tenancy, Pinecone namespaces, Qdrant payload-based partitioning; Postgres row-level security for pgvector).
- **Every cache is tenant-scoped**: semantic cache, embedding cache, prompt cache and conversation memory.
- **Separate keys**: per-tenant encryption keys for enterprise tenants, if required.
- **No cross-tenant learning** without consent. Fine-tuning or few-shot examples must not include other tenants' data.
- **Tests**: automated cross-tenant leakage tests in CI (tenant A queries for a unique canary string planted in tenant B, and must get zero results), plus logs of tenant_id alongside retrieved chunk tenant_ids with an alert on any mismatch.

Trade-off: per-tenant collections make deletion and residency simple ('delete tenant' = drop a collection) but hurt when you have 50k tenants; filtered shared collections scale better but make one missing filter a breach. That's why the filter lives in infrastructure, not in each developer's code."

**Follow-ups:**
- *Noisy neighbors?* → Per-tenant rate limits and quotas, and move large tenants to dedicated shards.
- *Tenant offboarding?* → Deletion across the vector DB, BM25 index, caches, logs and backups, verified with a deletion job report.
- *Shared public knowledge plus tenant data?* → Query two indexes and merge results; the public index is read-only and has no tenant data.

**Red flags:**
- Letting the model or the client pass `tenant_id`.
- Forgetting the semantic cache is a cross-tenant leak vector.
- "We'll add a filter in the prompt."

---

## Q7. Where does most RAG latency come from, and how do you reduce it?
*Source: interview posts*

**What the interviewer is testing:** Whether you measure per stage and know where the time actually goes.

**Strong answer:**

"I'd instrument every stage with tracing before guessing. A typical breakdown for a non-optimized pipeline:

| Stage | Typical | Fix |
|---|---|---|
| Query rewriting (LLM call) | 300–800 ms | Small model, or skip when the query is standalone |
| Query embedding | 20–100 ms | Local/small embedding model, cache |
| Vector + BM25 search | 10–100 ms | ANN tuning (ef_search), filters on indexed fields |
| Reranking (cross-encoder, 50 docs) | 100–400 ms | Rerank fewer candidates, smaller reranker, GPU |
| **LLM generation** | 1–5 s | Stream, shorter context, smaller model, prompt caching |
| Network / sequential orchestration | hidden 100s of ms | Parallelize independent calls |

So most wall-clock time is **LLM generation and extra LLM calls in the pipeline** (rewriting, grading, multi-query). The biggest user-visible metric is **time to first token**, which is driven by prompt length (prefill) and everything before generation.

Techniques:
- **Stream** the answer, so perceived latency ≈ TTFT.
- **Parallelize** BM25, vector search and any metadata lookups.
- **Cut context**: rerank to the top 3–5 chunks instead of stuffing 20. That shortens prefill and often improves quality.
- **Prompt/prefix caching** for the static system prompt.
- **Route**: easy queries skip rewriting and reranking.
- **Semantic cache** for repeated questions.
- **Co-locate** services in one region with the model endpoint.

But don't cut corners with fire-and-forget retrieval: if you start generating before retrieval returns, you get fast hallucinations.

Example: a support bot went from 4.8 s P95 to 1.9 s by removing a query-rewriting call for 70% of queries (a classifier decided), reranking 30 candidates instead of 100, dropping from 12 chunks to 5 in the context, and streaming."

**Follow-ups:**
- *P95 spikes but P50 is fine?* → Look for long-context requests, provider queueing, cold starts, or retries on timeouts.
- *Is reranking worth the latency?* → Measure it. Usually +5–15 points of precision for 100–200 ms, which is worth it when answers must be right.
- *Voice use case?* → The budget is about 1 s total, so pre-warm, keep the LLM small and fast, and speak a short acknowledgment while retrieving.

**Red flags:**
- "Use a faster vector DB" as the first answer.
- No per-stage measurement.
- Optimizing latency by skipping the wait for retrieval.

---

## Q8. Near-duplicate and conflicting documents are polluting retrieval. What do you do?
*[Added]*

**What the interviewer is testing:** Data quality ownership and conflict resolution beyond "tweak the retriever".

**Strong answer:**

"Two different problems:

**Near-duplicates** (e.g. five copies of the same policy PDF with minor differences) crowd out the top-K. You retrieve five versions of one fact and miss the second fact you needed.
- Dedupe at ingestion with MinHash/SimHash on the text (e.g. Jaccard > 0.9) and keep the canonical/latest copy.
- At query time, use MMR (Maximal Marginal Relevance) or cluster the results to diversify the top-K.

**Conflicts** (the 2022 policy says 20 days of leave, the 2024 one says 24) are worse, because the model may pick either or blend them.
- **Authority and recency metadata**: `source_authority` (official > wiki > email), `effective_date`, `status` (draft/approved/superseded).
- Filter out superseded and draft documents by default, and boost authoritative, recent ones in ranking.
- Instruct the model: if sources conflict, prefer the most recent authoritative one and **surface the conflict** with citations.
- **Report conflicts to content owners.** A nightly job that finds high-similarity chunks with contradictory numbers turns RAG into a data-quality tool.

Trade-off: aggressive dedupe can drop a legitimately different regional variant, so dedupe within the same `region` and `doc_type` scope.

Example: dedupe removed 31% of chunks in an internal wiki corpus, Recall@5 on multi-fact questions rose from 0.68 to 0.79, and embedding cost fell by the same 31%."

**Follow-ups:**
- *Who owns conflict resolution?* → Content owners. Engineering provides detection and routing, not policy judgments.
- *Can the model resolve conflicts itself?* → Only with metadata; without dates or authority it guesses.

**Red flags:**
- Increasing top-K to "catch everything".
- Letting the LLM silently pick between contradictory sources.

---

## Q9. What does the production layer around RAG look like (caching, re-indexing jobs, cost/latency tracking, PII)?
*[Added]*

**What the interviewer is testing:** Whether you can describe the whole system, not only the retrieval algorithm.

**Strong answer:**

```
Client → API Gateway (auth, rate limit, tenant)
        → RAG service
            ├─ Input guardrails (PII detect/redact, injection screen)
            ├─ Cache check (exact → semantic, tenant+permission scoped)
            ├─ Retriever (hybrid + ACL filter) → Reranker
            ├─ Prompt builder (grounding rules, citations, token budget)
            ├─ LLM gateway (routing, retries, fallback provider, cost tags)
            └─ Output guardrails (PII, citation check, refusal policy)
        → Tracing/metrics/logs (OpenTelemetry, Langfuse/LangSmith/Phoenix)

Offline: ingestion workers, CDC, re-index jobs (blue-green), reconciliation,
         eval pipeline on golden set, feedback → labeling queue
```

"The pieces I'd call out:
- **Caching**: exact-match cache (Redis) plus semantic cache with a strict similarity threshold (e.g. > 0.95), scoped by tenant and permissions, invalidated by `doc_id` tags on updates.
- **Re-indexing**: scheduled reconciliation plus event-driven incremental updates; blue-green for full rebuilds.
- **Cost and latency tracking**: per-request tokens in/out, model, cost, latency per stage, tagged by tenant/feature. Dashboards for P50/P95 and cost per resolved query.
- **PII**: redact before logging and before sending to third-party models where required; encrypt at rest; set retention policies on traces.
- **Quality monitoring**: sampled online faithfulness scoring, thumbs-up/down, 'no answer' rate and citation-click rate.
- **Runbooks**: provider outage, index corruption, cost spike.

The point is that RAG is search plus reasoning plus verification plus systems engineering. The retrieval algorithm is maybe 20% of the production work."

**Follow-ups:**
- *What is the one metric you'd put on the wall?* → Answer correctness on a sampled, judged set, alongside cost per resolved query.
- *Where do you log prompts safely?* → After PII redaction, with access controls and a retention of e.g. 30 days.

**Red flags:**
- A diagram that ends at "LLM → response".
- No mention of cost attribution or quality monitoring.

---

## Rapid-fire recap

- Staleness has two halves: missing new content and undeleted old content. The second one is worse.
- Volatile facts (prices, stock, status) come from live tools, not the vector index.
- Stable chunk IDs plus content hashes mean you only re-embed what changed.
- Chunk within sections; diff sections on re-ingestion; flip a version pointer atomically.
- Re-index blue-green behind an alias, with validation gates and a rollback window.
- 50M docs ≈ 500M chunks ≈ TBs of vectors: quantize, dedupe, shard, bulk-load.
- ACLs are filtered before retrieval, synced by events, and revocations get a deny-list.
- Tenant ID comes from the auth token and is injected by infrastructure, never by the LLM.
- Every cache is scoped by tenant and permissions.
- Most RAG latency is LLM calls: stream, parallelize, rerank to fewer chunks, cache prefixes.
- Dedupe with MinHash, diversify with MMR, resolve conflicts with authority and recency metadata.
- Never trade correctness for speed by generating before retrieval returns.
