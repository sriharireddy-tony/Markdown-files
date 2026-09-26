# 16 — Production System Design

**What interviewers probe here:** whether you can run an LLM system as a *product*, keeping it fast, affordable, observable and available when providers fail, and whether you can design an end-to-end system on a whiteboard. Strong candidates quantify cost and latency, name failure modes before they're asked, and design for change (model swaps, provider outages, growth).

[← Back to index](README.md)

1. [How do you handle latency, cost, token usage and failures in an LLM application?](#q1)
2. [How would you reduce cost without sacrificing answer quality?](#q2)
3. [What becomes the bottleneck when usage grows 100x?](#q3)
4. [Which types of cache exist, and which fits which scenario?](#q4)
5. [How do you monitor an AI application in production?](#q5)
6. [How do you detect model drift and data drift?](#q6)
7. [How do you attribute cost across teams sharing one LLM gateway?](#q7)
8. [Design a multi-region deployment with model failover and data-residency constraints.](#q8)
9. [Design graceful degradation for when the primary provider has a partial outage.](#q9)
10. [Migrate a production system from one model provider to another with zero downtime.](#q10)
11. [How do you handle provider rate limits at scale?](#q11)
12. [System design: an enterprise knowledge assistant for 50k employees.](#q12)
13. [System design: a customer-support AI with human escalation.](#q13)

---

<a id="q1"></a>
## Q1. How do you handle latency, cost, token usage and failures in an LLM application?
*Source: interview posts*

**What the interviewer is testing:** Whether you treat these as engineered properties with budgets, not afterthoughts.

**Strong answer:**

"I set explicit budgets per request type, measure them, and put controls in the gateway so every feature benefits.

**Latency:**
- Stream responses (TTFT is what users feel).
- Parallelize independent steps: retrieval + user-profile fetch + guardrail checks.
- Use smaller models for sub-steps (classification, query rewriting); cap output length.
- Prompt caching for static prefixes; keep services co-located in one region.
- Per-stage tracing, so I know whether retrieval, the reranker or the LLM is slow.

**Cost and token usage:**
- Log input, output and cached tokens per request, feature, tenant and model.
- Trim context (a reranker picks the top 3–5 chunks instead of 20), summarize long histories, avoid resending large tool outputs.
- Route to the cheapest adequate model; cache repeated answers.
- Hard `max_tokens` limits, per-tenant quotas and budgets, and alerts on spend anomalies.

**Failures:**
- Timeouts on every call. Retries with exponential backoff + jitter, only for retryable errors (429, 5xx, timeouts), never for 400s.
- Circuit breaker per provider/model, with fallback to a secondary model or provider.
- Validate outputs (schema, guardrails). On failure, retry with a repair prompt once, then fall back to a safe response.
- Idempotency keys for side-effecting actions.
- Graceful degradation: 'Search results without an AI summary' beats a 500 error.

Example budget for a support chat turn: P95 TTFT < 1.2 s, average cost < $0.004/turn, error rate < 0.5%, with dashboards and alerts on each."

**Follow-ups:**
- *Where do you enforce this?* → In a central LLM gateway (auth, quotas, retries, routing, logging), so each team doesn't reinvent it.
- *What's the most common hidden cost?* → Agents looping with ever-growing context, and verbose outputs.

**Red flags:**
- "We'll optimize cost later."
- Retrying everything, including invalid requests and non-idempotent actions.

---

<a id="q2"></a>
## Q2. How would you reduce cost without sacrificing answer quality?
*Source: interview posts*

**What the interviewer is testing:** Prioritized cost levers, with an eval as the guardrail.

**Strong answer:**

"First, **measure where the money goes**. Break cost down by feature, model, and input vs output tokens. Usually 2–3 features and one prompt pattern dominate. Then I apply levers from cheapest to riskiest, **re-running the eval set after each**, with an acceptance rule like 'no more than 1 point quality drop on the golden set, no drop on critical slices'.

| Lever | Typical saving (illustrative) | Quality risk |
|---|---|---|
| Prompt/prefix caching (static system prompt + tools first) | 30–90% of input cost on cached parts | None |
| Trim context: reranker → top-k 3–5, dedupe chunks | 30–60% of input tokens | Low; can improve quality |
| Cap / shorten outputs ('answer in ≤ 120 words', JSON instead of prose) | 20–50% of output cost | Low |
| Semantic / exact response cache for repeated questions | 10–40% of calls | Low if cache hits are validated |
| Model routing / cascade (small model first) | 40–70% | Medium; needs a per-route eval |
| Batch API for offline jobs | ~50% on those jobs | None; only latency changes |
| Summarize conversation history instead of resending it | 30–70% on long chats | Medium |
| Fine-tune / distill a small model for high-volume narrow tasks | 80–95% | Medium; needs data and upkeep |
| Self-host at high volume | Varies | Ops burden |

Example: a RAG assistant at $40k/month. Moving the 3k-token system prompt and tool definitions to a cached prefix, cutting retrieved chunks from 12 to 5 with a reranker, and routing FAQ-type queries to a small model brought it to about $14k/month. Faithfulness went slightly *up* because there was less noise in the context."

**Follow-ups:**
- *How do you prove quality didn't drop?* → An offline golden set + LLM judge calibrated against humans, then an online A/B test on CSAT, resolution and escalation rates.
- *What would you never cut?* → Guardrails and grounding checks in high-risk flows.

**Red flags:**
- "Use a cheaper model" as the only idea, with no eval.
- Optimizing without a cost breakdown.

---

<a id="q3"></a>
## Q3. What becomes the bottleneck when usage grows 100x?
*Source: interview posts*

**What the interviewer is testing:** Systems thinking about which limits appear, and in what order.

**Strong answer:**

"I'd walk the request path and ask what breaks first at 100x. Say we go from 1 to 100 req/s.

1. **Provider rate limits (RPM/TPM)** come first with API models. You need quota increases, multiple deployments or regions, a second provider, and a queue with priority.
2. **Cost scales linearly.** $10k/month becomes $1M. Caching, routing, distillation or self-hosting become mandatory, not optional.
3. **GPU capacity** if self-hosted: GPU availability, autoscaling lag, KV-cache limits on concurrency.
4. **Retrieval tier:** vector DB QPS and memory (HNSW indexes live in RAM), reranker GPU throughput, which is often a hidden bottleneck. You need sharding, replicas, and caching of embeddings and query results.
5. **Tail latency amplification:** agent flows with 5–10 sequential LLM calls, where P99 compounds.
6. **Stateful pieces:** conversation stores, Redis, WebSocket connection counts, DB connection pools.
7. **Ingestion/indexing:** 100x more documents or updates means embedding throughput, re-index time and freshness lag.
8. **Observability cost:** logging every prompt and response at 100x is a storage and privacy problem. Sample and tier retention.
9. **Humans in the loop:** a 2% escalation rate that was 20 tickets/day becomes 2,000.
10. **Long-tail quality:** 100x users bring inputs you never tested, and abuse or prompt-injection attempts rise.

So I'd load-test at 10x, fix the first bottleneck, repeat, and model cost per request before growth rather than after."

**Follow-ups:**
- *Which one bites teams most often?* → Provider rate limits and cost, then reranker/vector DB latency.
- *How do you plan for it?* → A capacity model (req/s × tokens → TPM, GPUs, $) reviewed monthly against growth forecasts.

**Red flags:**
- "Just add more servers."
- Ignoring external limits (provider quotas) and human processes.

---

<a id="q4"></a>
## Q4. Which types of cache exist (exact, semantic, prompt/prefix, retrieval, embedding), and which fits which scenario?
*Source: interview posts*

**What the interviewer is testing:** Understanding of caching layers and their correctness risks.

**Strong answer:**

| Cache | What's stored | Key | Best for | Risk |
|---|---|---|---|---|
| **Exact response cache** | Final answer | Hash(normalized prompt + model + params + context version) | Repeated identical requests, deterministic tasks (classification, extraction) | Low; needs invalidation when data or prompts change |
| **Semantic cache** | Final answer | Query embedding; hit if similarity > threshold (e.g. 0.95) | FAQ-style traffic ('how do I reset my password?') | **Wrong-answer hits** on similar-but-different questions ('cancel' vs 'don't cancel'); personalization and permission leaks |
| **Prompt / prefix (KV) cache** | Model's KV states for a prompt prefix | Exact prefix match | Long static system prompts, tool definitions, few-shot examples, shared documents | None for quality; requires a stable prefix ordering |
| **Retrieval cache** | Top-k chunk IDs for a query | Query (+ filters, index version) | Popular queries; saves vector DB + reranker latency | Staleness after re-indexing |
| **Embedding cache** | Vectors for texts | Hash(text + embedding model version) | Re-ingestion, repeated queries | Model version mismatch if the key lacks the model version |
| **Tool/API result cache** | External call results | Tool + args | Weather, prices, slow internal APIs | Freshness; TTL per tool |

"Scenario picks:
- **HR policy bot with repetitive questions:** semantic cache with a high threshold, scoped **per tenant and permission group**, invalidated when the source doc version changes. The key includes the doc index version.
- **Agent with a 5k-token tool manifest:** prompt caching. Put static content first and dynamic content last.
- **Personalized or account-specific answers** ('what's my balance?'): **no response cache**; cache only the retrieval of generic content.
- **High-stakes domains:** prefer an exact cache, or use the semantic cache only to *suggest*, then re-validate with a cheap judge.

Measure hit rate, **false-hit rate** (sample hits and judge them) and savings. A semantic cache with 30% hits and 2% wrong answers can be a net negative."

**Follow-ups:**
- *How do you pick the semantic threshold?* → Label a set of query pairs as same or different intent, then pick the threshold at an acceptable false-hit rate.
- *Where do you store it?* → Redis (with vector search) or the vector DB, with TTLs.

**Red flags:**
- A semantic cache shared across users with different permissions.
- No invalidation strategy.

---

<a id="q5"></a>
## Q5. How do you monitor an AI application in production (tracing, logging, dashboards, prompt drift, token usage)?
*Source: interview posts*

**What the interviewer is testing:** LLM observability beyond "we have logs".

**Strong answer:**

"Four layers: **system health, LLM operations, quality, and business outcomes**.

1. **System health (standard SRE):** request rate, error rate by type (timeouts, 429s, 5xx, validation failures), latency P50/P95/P99 per stage, saturation (queue depth, GPU KV usage).
2. **LLM operations:** tokens in/out/cached per request, cost per request, feature and tenant, model/version mix, retries and fallbacks triggered, tool-call counts and loop depth for agents, cache hit rates.
3. **Quality (the part most teams miss):**
   - Online evaluators on a sample (e.g. 5–10%): faithfulness/groundedness, relevance, refusal rate, format validity, safety flags.
   - Retrieval signals: top-1 similarity distribution, 'no relevant chunk' rate, citation coverage.
   - User signals: thumbs up/down, regenerations, copy events, escalation to a human, conversation abandonment.
4. **Business outcomes:** deflection or resolution rate, CSAT, time saved, conversion.

**Tracing:** every request gets a trace with spans for retrieval → rerank → LLM call(s) → tools → guardrails. Each span records the prompt template **version**, model version, token counts and latency. Tools: OpenTelemetry + Langfuse / LangSmith / Arize Phoenix / Datadog LLM Observability.

**Drift monitoring:**
- **Input drift:** embedding clusters of queries over time, new topics, language mix, query length.
- **Output drift:** answer length, refusal rate, sentiment, judge scores trending down.
- **Prompt drift:** prompts changed without a version bump; track the template hash per request.

**Privacy:** redact PII before storing traces, use role-based access to logs, and set retention limits.

**Alerts** on burn rates, not blips: e.g. 'faithfulness 7-day average down > 5%', 'refusal rate doubled', 'cost per request up 30% day over day'. And a weekly review: sample failures, label them, and add them to the golden set."

**Follow-ups:**
- *What's the single most useful quality signal?* → A calibrated groundedness judge on sampled traffic, plus explicit user feedback.
- *Log every prompt?* → Log metadata for all requests, full content for a sampled or consented subset with redaction.

**Red flags:**
- Only infra metrics and no quality monitoring.
- Logging raw PII-laden prompts indefinitely.

---

<a id="q6"></a>
## Q6. How do you detect model drift and data drift?
*Source: interview posts*

**What the interviewer is testing:** How drift shows up in LLM systems, where there's often no label feedback.

**Strong answer:**

"Three kinds matter:

1. **Data / input drift:** the query distribution changes (new product launch, new user segment, a language shift).
   - Detect by embedding queries, clustering them, and tracking cluster proportions over time. Alert on new clusters or share changes, and on population-stability or KL divergence between weekly distributions of intent labels, lengths and languages.
2. **Knowledge / corpus drift:** documents change or go stale; retrieval scores drop.
   - Detect via top-k similarity distributions, the 'no good context' rate, and doc freshness metrics (the age of cited sources).
3. **Model drift:** with API models, behavior can change when the vendor updates the model. With your own models, performance degrades as the world moves away from the training data.
   - Pin model versions (dated snapshots). Run a **daily canary eval** of a fixed golden set against production endpoints. Any score shift with no code change means model or provider drift.
   - Track output statistics: length, refusal rate, JSON validity, judge scores.

For classical ML components (classifiers, rankers) I'd also monitor prediction distribution shift and, where labels arrive later (e.g. ticket resolution), true accuracy with a delay.

**Response:** triage whether the drift is benign (a new topic that's handled fine) or harmful (quality down). Then add new examples to the eval set, update the corpus or prompts, or retrain. Drift monitoring is only useful if someone owns the alert."

**Follow-ups:**
- *There are no labels. How do you know quality dropped?* → Proxy signals: judge scores, user feedback, escalations, regenerations, and periodic human review of samples.
- *The vendor silently changed the model?* → Use pinned snapshot versions, and the canary eval catches it.

**Red flags:**
- Only talking about retraining classical models.
- Not pinning model versions.

---

<a id="q7"></a>
## Q7. How do you attribute cost across teams sharing one LLM gateway?
*Source: interview posts*

**What the interviewer is testing:** Platform thinking, FinOps and incentives.

**Strong answer:**

"**Make identity mandatory at the gateway.** Every request must carry a team, application, feature and environment, via per-app API keys or signed metadata headers, plus optionally a tenant or end-customer ID. Requests without them are rejected.

**Meter at the gateway, not the vendor invoice:**
- Log per request: input tokens, cached input tokens, output tokens, model, provider, region, latency and the tags.
- Price with a **versioned price table** (per model, per token type, batch vs real-time, cached discount), so historic costs stay correct when prices change.
- Self-hosted models: allocate GPU cost by token share or GPU-seconds per team, plus an idle-capacity share (a shared-cost pool distributed proportionally).
- Shared overhead (guardrail models, embeddings, gateway infrastructure): either allocate it proportionally, or show it as a platform line item.

**Reconcile** monthly against actual provider invoices. The gap (from retries, failed calls, rounding) should be under a few percent; investigate if not.

**Expose and enforce:**
- Dashboards per team and feature: cost/day, cost per request, top prompts by spend, cache hit rate.
- **Budgets and quotas** per team with alerts at 50/80/100%, and soft vs hard limits (hard for dev/test keys).
- Showback first, then chargeback once the numbers are trusted.
- Anomaly detection: a team whose spend triples overnight is usually an agent loop or a bug.

Example: a gateway (LiteLLM / Portkey / a custom Envoy-based one) tagging 40 apps. Within a month, one internal agent was found to be 35% of all spend, stuck in retry loops. Fixing it paid for the whole platform effort."

**Follow-ups:**
- *How do you attribute cost in multi-agent chains?* → Propagate the originating request's trace ID and tags to every sub-call.
- *Per-customer cost for pricing decisions?* → Add tenant tags, and compute gross margin per customer tier.

**Red flags:**
- Splitting the vendor bill evenly across teams.
- Tagging that's optional, so half of the spend goes untagged.

---

<a id="q8"></a>
## Q8. Design a multi-region deployment with model failover and data-residency constraints.
*Source: interview posts*

**What the interviewer is testing:** Availability design under legal constraints. Failover must not violate residency.

**Strong answer:**

"**Clarify first:** Which regions and regulations apply (e.g. EU data must stay in the EU, India data localization)? Does residency cover prompts, logs, embeddings and backups? What are the RTO/RPO and latency targets? Are the models API-based, self-hosted, or both?

**Architecture: residency zones, each independently complete:**
```
                 Global DNS / Anycast (geo + tenant routing)
                    │                            │
          ┌──── EU zone ────┐           ┌──── India zone ────┐
          │ eu-west  eu-cent│           │ ap-south-1 ap-south-2│
          │  Gateway  Gateway│          │  Gateway    Gateway  │
          │  RAG svc  RAG svc│          │  RAG svc    RAG svc  │
          │  Vector DB (repl)│          │  Vector DB (repl)    │
          │  LLM: EU endpoints│         │  LLM: IN endpoints   │
          └──────────────────┘          └──────────────────────┘
       Failover allowed ONLY within the same residency zone
```

**Key decisions:**
- **Tenant → home zone mapping** is stored in the tenant registry. The gateway enforces it: a request tagged EU can only call EU model endpoints. This is policy-as-code, with an audit log of every call's region.
- **Model failover chain per zone:** primary provider EU region A → same provider EU region B → secondary provider in the EU → self-hosted open model in the EU (degraded quality but compliant). **Never** fail over to a US endpoint for an EU tenant, even if that means degraded service. Legal constraints beat availability.
- **Data plane:** vector DB and document store replicated within the zone (active-active or active-passive). Embeddings count as personal data if they derive from it, so they stay in the zone.
- **Observability:** logs and traces stored in-zone. Only aggregated, non-personal metrics go global.
- **Control plane** (configs, prompt versions, model registry) can be global, as long as it contains no customer data.
- **Consistency:** prompt and model versions rolled out per zone with canaries. Each zone has its own eval run, since provider models can differ by region.
- **Health checks and circuit breakers** per endpoint. Failover triggered by error rate or latency SLO breaches, with automatic failback after stability.

**Trade-offs:** duplicated infrastructure cost per zone; some models aren't available in every region, so capability can differ per zone and needs a documented feature matrix."

**Follow-ups:**
- *A tenant has users in both zones?* → The data follows the tenant's contractual home. Or split tenants by data classification if contracts allow.
- *How do you test failover?* → Regular game days: kill the primary endpoint in staging and production-like environments and measure RTO and quality.

**Red flags:**
- A global failover that silently crosses residency boundaries.
- Forgetting that logs, embeddings and backups are also data.

---

<a id="q9"></a>
## Q9. Design graceful degradation for when the primary provider has a partial outage.
*Source: interview posts*

**What the interviewer is testing:** Resilience patterns, and product judgment about which degraded experience is acceptable.

**Strong answer:**

"A partial outage is the hard case: elevated errors, 30-second latencies, or one model down while others work. Everything isn't cleanly down.

**Detection:** per-model/per-endpoint health from real traffic (error rate, P95 latency, timeouts), not just the vendor status page. A **circuit breaker** opens when, for example, the error rate is > 20% over 1 minute or P95 is > 3× baseline. It half-opens periodically to probe recovery.

**Degradation ladder (per feature, decided in advance with product):**
```
Level 0  Normal: primary model
Level 1  Same provider, alternate region/deployment
Level 2  Secondary provider, equivalent model (prompt adapter per model)
Level 3  Smaller/self-hosted model with reduced features (shorter answers, no tools)
Level 4  Non-generative fallback: search results, cached answers, FAQ links, templates
Level 5  Honest message + queue: 'AI assistant is limited right now; we'll email your answer'
```

**Techniques:**
- **Tight timeouts + hedging:** for latency-critical calls, send a backup request to the secondary after P95 time and take whichever returns first. Watch cost.
- **Load shedding by priority:** keep paying customers and critical flows; pause batch jobs, evals and background summarization.
- **Feature flags** to switch features into degraded mode instantly, without deploying.
- **Retry budgets:** cap retries at ~10% of traffic so retries don't amplify the outage (retry storms).
- **Queue async work** for replay after recovery, with idempotency keys.
- **Tell the user:** a UI banner and adjusted expectations.

**Prerequisites:** fallback models must be **pre-evaluated** on the golden set, and their prompts tested. The first time you run your fallback prompt shouldn't be during an outage. Run regular chaos drills."

**Follow-ups:**
- *How do you avoid flapping between providers?* → Hysteresis: open quickly, close only after N successful probes over a window.
- *Does the secondary need warm capacity?* → Yes. Quotas must already be provisioned and tested at the expected failover load.

**Red flags:**
- "We retry until it works."
- Failing over to an untested model and prompt.

---

<a id="q10"></a>
## Q10. Migrate a production system from one model provider to another with zero downtime.
*Source: interview posts*

**What the interviewer is testing:** Safe-migration discipline: abstraction, evaluation, staged rollout, rollback.

**Strong answer:**

"I'd run it like a database migration, in phases, with a rollback at every step.

**Phase 0: Abstraction.** All calls go through a gateway or internal SDK with a provider-agnostic interface (messages, tools, structured output, streaming). No feature calls a vendor SDK directly. If that's not already true, fix it first.

**Phase 1: Offline parity.**
- Inventory every prompt, feature and model-specific behavior: tool-calling format, JSON mode, system prompt handling, token limits, stop sequences, tokenizer differences (which affect cost and context budgets).
- Port the prompts. They rarely transfer 1:1, so expect to re-tune them.
- Run the **full eval suite** per feature on both providers: quality, format validity, refusal behavior, latency and cost. Also check embeddings: **if the embedding model changes, you must re-index** (a new index built in parallel).
- Define go/no-go criteria per feature.

**Phase 2: Shadow traffic.** Mirror a sample of real production requests to the new provider asynchronously. Compare outputs with a judge and diff tooling, plus latency and errors. Users see nothing.

**Phase 3: Canary.** Route 1% → 5% → 25% → 50% → 100% by feature (lowest-risk first), with feature flags. Monitor online metrics: CSAT, escalation, thumbs-down, error rate. Automatic rollback triggers on SLO breach.

**Phase 4: Cutover and cleanup.** Keep the old provider as the **fallback** for a few weeks (it doubles as graceful degradation), then decommission. Update docs, runbooks and cost dashboards.

**Details that bite:**
- Conversation continuity: a session that started on provider A can continue on B. Store history in a provider-neutral format.
- Caches keyed by model must be separated or invalidated.
- Contracts: data-processing agreements, regions and rate-limit quotas on the new provider provisioned *before* the ramp.

Example: moving 12 features took about 6 weeks. Two features needed prompt rewrites because the new model was more verbose; we added explicit length constraints, and the cost ended 22% lower. (Illustrative.)"

**Follow-ups:**
- *What if the new model is better on average but worse on one slice?* → Keep that slice on the old model via routing until it's fixed. Migration doesn't have to be all or nothing.
- *How do you migrate embeddings?* → Build a dual index, backfill, dual-read with evals, switch the read path, then delete the old index.

**Red flags:**
- A big-bang switch with a config change on Friday.
- Assuming prompts and embeddings are portable.

---

<a id="q11"></a>
## Q11. How do you handle provider rate limits at scale?
*[Added]*

**What the interviewer is testing:** Practical throughput engineering against external quotas.

**Strong answer:**

"Limits are usually **RPM and TPM** (tokens per minute), per model, per region, per key.

- **Know your demand in TPM:** peak req/min × average tokens per request. Request quota increases or provisioned throughput ahead of launches.
- **Client-side rate limiting at the gateway:** token-bucket per model/deployment, sized to the quota, so we queue locally instead of hammering the provider into 429s.
- **Priority queues:** interactive requests first, batch or background jobs last. Batch jobs go to the batch API where it exists.
- **Retry 429s correctly:** honor `Retry-After`, exponential backoff with jitter, and a retry budget.
- **Spread load** across multiple deployments, regions (within residency rules) or providers.
- **Reduce tokens per request:** caching and trimming effectively raise the ceiling.
- **Per-tenant fairness:** a per-tenant quota so one customer's bulk job can't starve everyone.
- **Provisioned throughput / reserved capacity** for predictable base load, with pay-as-you-go for bursts."

**Follow-ups:**
- *How do you size provisioned throughput?* → Cover P50–P75 of baseline load with reserved capacity, and handle bursts on-demand.

**Red flags:**
- Handling 429s with immediate tight-loop retries.

---

<a id="q12"></a>
## Q12. System design: an enterprise knowledge assistant for 50k employees.
*[Added]*

**What the interviewer is testing:** An end-to-end design covering requirements, architecture, security, evaluation and scale.

**Strong answer:**

"**Clarify requirements:**
- Sources: Confluence, SharePoint, Google Drive, Jira, HR policies, Slack. Roughly 5M documents.
- Users: 50k employees; perhaps 20% are daily active and send 5 queries each → about 50k queries/day, a peak of ~5 req/s.
- Non-functional: permission-aware answers (**the hardest requirement**), citations, P95 < 4 s to the full answer with streaming, SSO, audit logs, data residency, and freshness within ~15 minutes.

**Architecture:**
```
Connectors (webhooks + incremental sync) ──► Ingestion queue (Kafka)
   ──► Parse (layout-aware, OCR, tables) ──► Chunk (structure-aware) ──► Embed (batch GPU)
   ──► Index: vector DB + BM25, with metadata {doc_id, version, ACL groups, source, updated_at}

User (SSO/OIDC) ──► API gateway ──► Assistant service
   1. Resolve user identity + group memberships (cached, short TTL)
   2. Query rewrite (conversation-aware), intent routing (policy Q&A / people lookup / Jira)
   3. Hybrid retrieval WITH ACL filter at query time → RRF → cross-encoder rerank (top 5)
   4. Prompt with grounding rules + citations → LLM (streaming)
   5. Output checks: citation present, PII policy, groundedness sample-check
   ──► Response + citations (doc, section, link)
Side: trace store, eval pipeline, feedback, cost dashboards
```

**Key decisions:**
- **Permissions:** store ACL groups on each chunk and filter at retrieval time (pre-filtering in the vector DB). Sync permission changes quickly, via webhooks plus a periodic reconciliation job. Never rely on the LLM to hide content. Deletions and revocations propagate as tombstones immediately.
- **Freshness:** incremental, versioned re-indexing. Swap versions only after a new version is fully indexed.
- **Hybrid search**, because enterprise queries are full of acronyms, ticket IDs and names, where BM25 wins.
- **Refusal behavior:** 'I couldn't find this in documents you have access to' plus suggested owners.
- **Model:** mid-tier model with prompt caching. Estimated cost: 50k queries × ~3k input + 400 output tokens ≈ 150M input / 20M output tokens per day. At illustrative mid-tier prices that's roughly $10–20k/month before caching.
- **Evaluation:** a golden set of 500 questions per department with ground-truth docs; retrieval Recall@5, faithfulness and citation accuracy; a permission-leak test suite (queries from users who *shouldn't* see docs, where the pass criterion is zero leaks).
- **Rollout:** pilot with one department, measure deflected tickets and time saved, then expand."

**Follow-ups:**
- *What's the biggest risk?* → Permission leakage, then stale answers. Both are handled at the data layer, not in the prompt.
- *How do you handle 'who owns X?' questions?* → Route to a people/directory tool, not document RAG.

**Red flags:**
- Ignoring document-level permissions.
- No evaluation or feedback loop.

---

<a id="q13"></a>
## Q13. System design: a customer-support AI with human escalation.
*[Added]*

**What the interviewer is testing:** Designing autonomy boundaries, handoff and success metrics.

**Strong answer:**

"**Requirements:** channels (chat, email, possibly voice), knowledge base + order/account APIs, and actions like refunds or address changes. Target: resolve 40–60% of contacts automatically with CSAT at least as good as human agents, and fast, context-rich escalation for everything else.

**Architecture:**
```
Customer ─► Channel adapter ─► Conversation service (state in DB/Redis)
   ─► Intent + risk classifier (small model / rules)
        ├─ FAQ/policy  ─► RAG answer with citations
        ├─ Account task ─► Agent with scoped tools (read order, track shipment)
        ├─ Sensitive / high-risk (legal threat, fraud, self-harm, VIP, angry) ─► Human immediately
        └─ Action with side effects (refund > ₹2,000) ─► Human approval queue
   ─► Guardrails (PII masking, policy compliance, tone)
   ─► Response  |  Escalation ─► Ticketing (Zendesk/Freshdesk) with summary
```

**Escalation design (the core of the question):**
- **Triggers:** low confidence or 'no grounded answer', explicit request for a human, sentiment/frustration detection, repeated failed turns (e.g. 2 misunderstandings), policy categories, action limits.
- **Handoff package:** a conversation summary, detected intent, customer and order details, what the AI already tried, and suggested next steps. The customer never repeats themselves.
- **Agent assist:** when a human takes over, the AI drafts replies they can edit. This is also a data source for improving the bot.
- **Authentication** before any account action (OTP or a logged-in session). Tools are scoped to *that* customer's ID only.
- **Idempotent actions** with audit logs. Refunds are capped per policy, with human approval above the cap.

**Metrics:** automated resolution rate (confirmed by no recontact within 72 h, not just 'conversation ended'), CSAT per path, escalation rate and escalation quality (did the human need to re-ask?), handle time, cost per contact, and a hallucination/compliance audit on a weekly sample.

**Rollout:** start in agent-assist mode, then automate the top 10 intents behind a confidence threshold, raising autonomy as the evals and CSAT prove it out."

**Follow-ups:**
- *How do you avoid the bot trapping customers?* → Always offer a clear route to a human, and cap the number of bot turns before offering escalation.
- *How do you learn from escalations?* → Cluster escalated conversations, find top missing-knowledge or intent gaps, fix them, and add cases to the eval set.

**Red flags:**
- Measuring success as 'conversations not escalated' (which rewards trapping users).
- Letting the agent execute refunds without authentication, limits or audit.

---

## Rapid-fire recap

- Set per-request budgets for latency, cost and error rate, and enforce them in a central LLM gateway.
- Cost levers from cheapest to riskiest: prompt caching → trim context → cap outputs → cache answers → route models → distill. Re-run evals after each.
- At 100x: provider TPM limits and cost bite first, then the retrieval/reranker tier and tail latency.
- A semantic cache must be scoped per tenant and permission, with a measured false-hit rate.
- Monitor four layers: system health, LLM ops, quality (sampled judges + user signals) and business outcomes.
- Pin model versions, and run a daily canary eval to detect provider drift.
- Cost attribution: mandatory tags at the gateway, a versioned price table, reconciliation, showback then chargeback.
- Multi-region: fail over only within a residency zone. Logs, embeddings and backups are data too.
- Graceful degradation: a pre-agreed ladder, circuit breakers, retry budgets, and pre-evaluated fallbacks.
- Provider migration: abstraction → offline eval → shadow → canary by feature → keep the old provider as fallback.
- Enterprise assistant: permission filtering at retrieval time plus freshness are the hardest parts.
- Support AI: escalation triggers, a rich handoff package, and resolution measured by no recontact.
