# 05 — Advanced RAG

**What interviewers probe here:** whether you know the advanced patterns (CRAG, Self-RAG, GraphRAG, HyDE, RAG Fusion, contextual retrieval, agentic RAG) as **trade-offs**, not buzzwords. Expect to discuss what each costs in latency and money, when it's worth it, how you'd A/B test it, and how you'd stop loops.

[← Back to index](README.md)

## Questions

1. [Overview of advanced RAG techniques](#q1)
2. [How RAG Fusion affects latency](#q2)
3. [Designing a cost-efficient CRAG grader](#q3)
4. [Self-RAG vs CRAG production cost](#q4)
5. [Contextual Retrieval for 10M documents](#q5)
6. [GraphRAG vs standard RAG](#q6)
7. [When multi-hop RAG should stop](#q7)
8. [A/B testing HyDE](#q8)
9. [Iteration limits for Agentic RAG](#q9)
10. [Do agents and tools improve a RAG system?](#q10)
11. [Do we still need RAG with 1M-token context?](#q11)
12. [Parent-document, small-to-big and sentence-window retrieval](#q12)

---

<a id="q1"></a>
## Q1. Give an overview of advanced RAG: Multi-Query, RAG Fusion, HyDE, Self-RAG, CRAG, Agentic RAG, GraphRAG. What problem does each solve?
*Source: interview posts*

**What the interviewer is testing:** whether you can map each technique to the failure mode it fixes, instead of reciting a list.

**Strong answer:**

"I group them by *which stage they fix*:

| Technique | Fixes | How | Cost |
|---|---|---|---|
| **Multi-Query** | Ambiguous or poorly worded queries (low recall) | LLM generates N rephrasings, retrieve each, union | N retrievals + 1 LLM call |
| **RAG Fusion** | Same as multi-query, better ranking | Multi-query + RRF over the result lists | Same + fusion (cheap) |
| **HyDE** | Query and documents look different (short question vs long technical doc) | LLM writes a hypothetical answer; embed *that* for search | 1 LLM generation before retrieval |
| **CRAG** (Corrective RAG) | Retrieved docs are irrelevant or incomplete | A grader scores the retrieved docs; if poor → rewrite the query or use a fallback (web search, other index); strip irrelevant parts | Grader call(s) + conditional fallback |
| **Self-RAG** | The model retrieves when unnecessary, or generates unsupported claims | The model decides *whether* to retrieve and self-critiques relevance, support and usefulness (originally a fine-tuned model with reflection tokens) | Multiple critique steps per answer |
| **Agentic RAG** | Multi-step and multi-source questions | An agent plans, chooses tools (vector DB, SQL, web), and iterates | Unbounded unless capped |
| **GraphRAG** | Multi-hop and global "summarize the themes" questions over relationships | Extract an entity/relation graph (plus community summaries); traverse or query the graph | Very expensive indexing (LLM over the whole corpus) |

My rule is to reach for them in order of cost. First fix parsing, chunking, hybrid search and reranking. Then add query-side techniques (multi-query, HyDE) if recall is the problem. Then add CRAG-style grading if precision or irrelevant context is the problem. Use agentic RAG or GraphRAG only for query types that truly need multi-step or relational reasoning, and route only those queries there."

**Follow-ups:**
- *Which gave you the most lift in practice?* → Usually hybrid + reranker (basic). Among the advanced ones, query decomposition for comparison questions, and a CRAG-style grader with a refusal path.
- *Can you combine them?* → Yes, but latency stacks. Use a router so only hard queries take the expensive path.

**Red flags:**
- Listing techniques without the failure mode each fixes.
- Recommending GraphRAG or agents before basic retrieval is solid.

---

<a id="q2"></a>
## Q2. How does RAG Fusion affect latency?
*Source: interview posts*

**What the interviewer is testing:** whether you can reason about the critical path and optimize it.

**Strong answer:**

"RAG Fusion adds three things to the critical path:

```
[LLM: generate N queries] -> [N retrievals] -> [RRF] -> [rerank] -> [LLM answer]
     +300-800 ms            +parallel 30-100 ms  ~1 ms  larger candidate set
```

1. **Query generation:** one extra LLM call, typically 300–800 ms on a mid-size model. That's the **biggest cost**.
2. **N retrievals:** run in **parallel**, so the latency is roughly the slowest single retrieval, not N×. But you get N× the vector DB load (QPS), which matters for capacity.
3. **RRF:** negligible.
4. **Reranking:** the union of N lists is larger, for example 4 × 20 = 80 candidates instead of 20, so cross-encoder time grows about linearly.
5. **Token cost:** only if you pass more chunks to the LLM. Keep the final context size the same.

Typical net effect: **+0.5–1.2 s P50** and a noticeably higher P95.

**How I'd reduce it:**
- Generate the queries with a small, fast model (or a fine-tuned small model), capped at 3 queries and short outputs.
- Run the original query's retrieval **concurrently** with query generation, and merge when the generated queries arrive.
- Make it **adaptive**: only fan out when first-pass retrieval confidence (top reranker score) is low.
- Cache generated queries for frequent or normalized queries.
- Cap the rerank candidates (dedupe the union first).

I'd only keep it if the golden set shows a recall or answer gain that's worth it for the product, for example +6 points of accuracy for +600 ms in an async research tool, but not in a voice agent."

**Follow-ups:**
- *What about vector DB cost?* → N× query load. Budget replicas accordingly, or apply fusion to a small fraction of traffic via the adaptive trigger.

**Red flags:**
- "Latency multiplies by N" (ignoring parallelism).
- Ignoring the query-generation LLM call, which dominates.

---

<a id="q3"></a>
## Q3. How would you design a cost-efficient CRAG grader?
*Source: interview posts*

**What the interviewer is testing:** practical cost engineering. Can you get corrective behavior without an expensive LLM call per chunk?

**Strong answer:**

"CRAG's grader decides, for the retrieved set: **correct** (use it), **ambiguous** (refine or supplement it), or **incorrect** (fall back: rewrite the query, search another source, or refuse). The naive version asks a large LLM to grade every chunk, which means 5–10 LLM calls per query. That's too expensive.

**A cost-efficient design, as a cascade:**

1. **Tier 0, free signals:** the reranker score is already computed. If the top-1 score is above a high threshold, it's **correct** and there's no LLM grading. If all scores are below a low threshold, it's **incorrect**, so go straight to the fallback. Only the middle band continues. On most corpora that's 60–80% of traffic decided at zero cost.
2. **Tier 1, small grader:** for the ambiguous band, use a small model: a fine-tuned cross-encoder or a small LLM (for example a 1–8B model, or a cheap API tier). Grade **all chunks in one batched call** with structured output `[{chunk_id, relevant: bool}]`, not one call per chunk.
3. **Tier 2, expensive path (rare):** query rewrite + second retrieval, or web search, only for 'incorrect' cases.

**Calibration:** label 500 query–chunk pairs, then set thresholds to target, for example, 95% precision on 'correct'. Distill the large-LLM judgments into the small grader. Monitor the grader's agreement with a periodic large-LLM audit on 1% of traffic.

**Cost math example:** 1M queries a month. The naive approach is 1M × 8 chunks × a large-LLM grade. The cascade is 30% × 1 small batched call + 5% fallback, which is roughly a **10–20x cost reduction** at nearly the same answer quality.

Also, strip irrelevant sentences from 'correct' chunks (knowledge refinement) only when the context budget matters."

**Follow-ups:**
- *What's the fallback when it's 'incorrect'?* → Depending on the product: web search (public domains), a different index, asking a clarifying question, or an honest refusal.
- *Latency?* → Tier 0 adds 0 ms, Tier 1 about 150–300 ms, and only a minority takes Tier 2.

**Red flags:**
- One GPT-4-class call per chunk.
- No thresholds calibrated on labeled data.

---

<a id="q4"></a>
## Q4. Self-RAG vs CRAG: what are the production cost implications?
*Source: interview posts*

**What the interviewer is testing:** understanding of where each spends compute, and operational complexity.

**Strong answer:**

"They put the correction in different places:

| | CRAG | Self-RAG |
|---|---|---|
| Where it corrects | **Retrieval side**: grades retrieved docs before generation | **Generation side**: decides whether to retrieve, then critiques its own output for relevance, support and usefulness |
| Extra calls | Grader (can be cheap/small) + occasional fallback | Retrieve-decision + per-segment critique + possibly regenerating; multiple passes |
| Model requirement | Any LLM; grader is swappable | Original paper: a **fine-tuned model** with reflection tokens; prompting-based approximations are less reliable |
| Latency | +0.2–0.5 s typical, bounded | Higher and **variable** (depends on how many critique/regenerate loops) |
| Cost predictability | Good: a bounded cascade | Worse: long-tail cost from loops |
| Ops complexity | Low–medium | High: hosting or fine-tuning a specialized model, tuning critique thresholds |

**Production implications:**
- CRAG is **cheaper and more predictable**. You can build it with a threshold cascade (see Q3) and swap the models freely.
- Self-RAG can save cost on queries that don't need retrieval at all (it skips retrieval), and it catches unsupported claims post-hoc. But the critique loops create long-tail latency, and you're tied to a model you must fine-tune and maintain.
- A common hybrid: **retrieval router** (skip retrieval for chit-chat) + **CRAG grader** + **one post-generation faithfulness check** (claim vs context) with at most one regeneration. That gets most of Self-RAG's benefit with bounded cost.

I'd pick CRAG-style for most enterprise Q&A, and consider Self-RAG-style critique only where unsupported claims are very costly and the latency budget is generous."

**Follow-ups:**
- *How do you bound Self-RAG loops?* → A max of 1–2 regenerations, then answer with a caveat or refuse.

**Red flags:**
- Treating them as interchangeable.
- Not mentioning the fine-tuned-model requirement or variable latency.

---

<a id="q5"></a>
## Q5. How would you implement Contextual Retrieval for 10M documents?
*Source: interview posts*

**What the interviewer is testing:** whether you know the technique, and whether you can engineer the cost at scale.

**Strong answer:**

"**Contextual Retrieval** (Anthropic's technique) fixes chunks that lose meaning out of context. 'Revenue grew 3% over the prior quarter' doesn't say which company or quarter. For each chunk, an LLM generates a short 50–100 token context from the **full document** ('This chunk is from ACME's Q2 2023 10-Q, discussing revenue...'), and that context is **prepended to the chunk before both embedding and BM25 indexing** (contextual embeddings + contextual BM25). Anthropic reported large reductions in retrieval failures, and larger still when combined with a reranker.

**The challenge at 10M documents:** say 10M docs × ~20 chunks = **200M LLM calls**, each needing the whole document in the prompt.

**Engineering plan:**
1. **Prompt caching is the key cost lever.** Structure each call as `[full document] + [chunk i]`. The document prefix is cached, so the chunks of the same document reuse it. Process all chunks of one document **consecutively** so the cache stays warm. Cached input tokens are billed at a fraction of the normal rate.
2. **Small, cheap model** (a Haiku-class model or a self-hosted 7–8B), with a max of about 100 output tokens.
3. **Batch APIs** or offline batch inference for a further discount, since this isn't latency-sensitive.
4. **Long documents** larger than the context window: use the section plus the document title and summary instead of the full document.
5. **Prioritize:** run the high-value corpora first, and skip chunks that are already self-contained (FAQs, entity rows), detected by heuristics.
6. **Pipeline:** queue per document → workers → idempotent writes keyed by `(chunk_id, doc_version, prompt_version)` → re-embed. Checkpoint progress and retry failures.
7. **Incremental:** only regenerate context for changed documents. Store the context separately so you can re-embed without regenerating it.

**Rough cost framing:** estimate tokens as docs × (doc tokens cached once + chunks × (chunk + output tokens)), then apply cached and batch pricing. Validate on a 10k-document pilot: measure the recall@20 lift on the golden set before committing the full budget."

**Follow-ups:**
- *Does it increase index size?* → Slightly: about 50–100 tokens per chunk of text. The vector dimension is unchanged.
- *Self-hosted alternative?* → vLLM with prefix caching gives a similar prefix-reuse benefit.

**Red flags:**
- Not knowing that the context is prepended for both embedding and BM25.
- Ignoring the 200M-call cost problem, or not knowing prompt caching.

---

<a id="q6"></a>
## Q6. GraphRAG vs standard RAG: when does graph-based retrieval beat vector search?
*Source: interview posts*

**What the interviewer is testing:** whether you know what graphs add (relationships, multi-hop, global questions) and what they cost.

**Strong answer:**

"Vector search answers 'which chunks look like this question'. It's weak when the answer requires **connecting facts spread across documents**, or **aggregating across the whole corpus**.

**GraphRAG** (for example Microsoft's GraphRAG, or a knowledge graph in Neo4j) uses an LLM to extract entities and relationships into a graph, and often builds **community summaries** by clustering.

**Where graphs win:**
- **Multi-hop:** 'Which suppliers of our top-selling product had compliance issues last year?' This needs product → supplier → audit relationships.
- **Global/sensemaking questions:** 'What are the main themes of complaints this quarter?' Community summaries answer this, where top-K chunks can't.
- **Relationship-heavy domains:** org charts, fraud networks, drug interactions, dependencies between services.
- **Explainability:** you can show the path of facts.

**Where standard RAG wins:**
- Direct lookups ('what's the refund window'), which are most enterprise queries.
- Fast-changing corpora, because graph extraction is expensive to keep fresh.
- Budget: indexing means LLM calls over the whole corpus, often 10–100x the cost of embedding, and the extraction has errors (duplicate entities, missed relations).

**My approach:** start with hybrid RAG. If query analysis shows a meaningful share of multi-hop or global questions, build a graph for the **relationship-rich subset**, and route those query types to it. Keep both (hybrid 'graph + vector'): use the graph to find related entities, then vector search for the supporting text.

An alternative that's often cheaper: if the relationships already exist in structured systems (CRM, ERP), query them via SQL or a graph DB directly instead of extracting them from text."

**Follow-ups:**
- *Entity resolution issues?* → Normalize and merge entities (with rules plus embedding similarity), and keep canonical IDs. This is the hardest part of graph quality.
- *How to evaluate?* → A separate golden set of multi-hop and global questions; compare against a hybrid baseline.

**Red flags:**
- "GraphRAG is just better."
- No awareness of indexing cost or extraction errors.

---

<a id="q7"></a>
## Q7. How do you decide when a multi-hop RAG process should stop?
*Source: interview posts*

**What the interviewer is testing:** loop control, using both sufficiency signals and hard safety limits.

**Strong answer:**

"I use **two layers of stopping**: a semantic stop (we have enough) and a hard stop (budget).

**Semantic stop conditions:**
1. **Sufficiency check:** after each hop, a small model is asked 'given the question and the evidence so far, can it be fully answered? Which sub-question is still missing?' It returns a structured `{answerable: bool, missing: [...]}`. If answerable, stop and generate.
2. **Plan completion:** if I decomposed the question into sub-questions up front, stop when every sub-question has supporting evidence.
3. **No new information:** if a hop retrieves chunks already seen (high overlap by chunk ID) or the reranker scores are all low, further hops are unlikely to help. Stop.
4. **Repeated query:** if the next generated query is nearly identical to a previous one (by embedding similarity), that's a loop signal. Stop.

**Hard stops (always enforced in code, not in the prompt):**
- Max hops (for example 3–4; most real questions need 2–3).
- Max total tokens and cost per request.
- Wall-clock timeout (for example 10 s for interactive use).

**When a hard stop triggers:** answer with what's supported, state what couldn't be found, and log the trace for analysis. Don't loop silently or make things up.

I tune max hops from data: in one system, 92% of successful answers needed 3 hops or fewer, and hops 4–5 added under 1% accuracy at a 40% latency increase, so the cap was 3."

**Follow-ups:**
- *Who decides "sufficient": the same LLM?* → It can be, but a separate grader prompt with a structured output is more reliable than the generator judging itself.

**Red flags:**
- Only "the LLM decides when it's done", with no hard caps.
- No plan for what to return when the budget runs out.

---

<a id="q8"></a>
## Q8. How would you A/B test HyDE?
*Source: interview posts*

**What the interviewer is testing:** experimental design, with offline and online metrics and guardrails, applied to a retrieval change.

**Strong answer:**

"**HyDE** (Hypothetical Document Embeddings): the LLM writes a hypothetical answer to the query, and I embed that instead of, or alongside, the query. It helps when short queries and long documents live in different parts of embedding space. It can hurt when the hypothetical answer is confidently wrong and steers retrieval off course.

**Step 1: offline first (cheap, fast):**
- Golden set of 300–1,000 queries with labeled relevant chunks, **segmented by query type** (short or vague, keyword/ID, long).
- Arms: A = query embedding; B = HyDE embedding; C = both (fuse with RRF).
- Metrics: recall@20, MRR, and final answer accuracy and faithfulness, plus added latency and cost.
- Look at the segments. HyDE often helps vague queries and hurts ID lookups, which suggests **routing** rather than all-or-nothing.

**Step 2: online A/B (if offline is promising):**
- Randomize **by user or session** (not by request) so users get a consistent experience. Start with 10% of traffic.
- **Primary metric:** task success, meaning thumbs-up rate, answer acceptance, or no follow-up rephrase within 60 s, or resolution without escalation in support.
- **Secondary:** retrieval click-through on citations, LLM-judge faithfulness on a sample.
- **Guardrails:** P95 latency (HyDE adds an LLM call), cost per query, and refusal and hallucination rates.
- **Sample size:** compute it up front from the baseline success rate and the minimum detectable effect. For example, detecting a 2-point lift on a 70% base rate needs roughly 8–9k sessions per arm at 80% power.
- Run for at least a full weekly cycle; check for novelty effects.

**Decision rule, written before the test:** ship if the primary metric improves significantly and P95 latency stays within +400 ms. Otherwise, ship only for the segment where it helps."

**Follow-ups:**
- *How do you reduce HyDE's latency?* → A small model, short hypothetical outputs (about 100 tokens), and running the plain query retrieval in parallel.

**Red flags:**
- Launching to 50% of traffic without an offline eval.
- Randomizing per request, or having no guardrail metrics.

---

<a id="q9"></a>
## Q9. What iteration limits would you use for Agentic RAG?
*Source: interview posts*

**What the interviewer is testing:** concrete, defensible numbers, and enforcement in code.

**Strong answer:**

"I set limits at several levels, **enforced by the orchestrator, not the prompt**:

| Limit | Interactive chat | Async research task |
|---|---|---|
| Max agent iterations (LLM turns) | 5–8 | 20–30 |
| Max retrieval calls | 3–5 | 15 |
| Max identical tool calls (same tool + same args) | 1 (dedupe/cached) | 1 |
| Max tokens per request | ~30–50k | ~300k |
| Max cost per request | e.g. $0.05 | e.g. $1 |
| Wall-clock timeout | 15–30 s | 10 min, with checkpoints |

**How I choose them:** from traces. Plot the distribution of iterations for *successful* runs. If P95 successful runs use 4 iterations, set the cap at around 6–8. Runs beyond that are almost always loops. Then verify on the eval set that the cap doesn't reduce accuracy.

**Graceful behavior at the limit:**
- At about 80% of the budget, inject a system note: 'Budget nearly exhausted. Answer with the evidence you have.'
- At the limit, force a final answer from the gathered evidence with an explicit caveat, or escalate to a human. Never fail silently.

**Loop detectors in addition to the caps:** repeated tool+args hashes, no new chunk IDs in the last 2 retrievals, and the model's thought repeating.

Log the reason for every stop (`finished`, `max_iter`, `budget`, `timeout`) and alert if the non-`finished` rate rises. That's an early signal of prompt or tool regressions."

**Follow-ups:**
- *In LangGraph?* → `recursion_limit` in the config, plus custom counters in state for tool-specific caps.

**Red flags:**
- "The model knows when to stop."
- One global limit with no per-tool or cost caps.

---

<a id="q10"></a>
## Q10. Will integrating agents and tools into a RAG system improve it? Why, and how?
*Source: interview posts*

**What the interviewer is testing:** balanced judgment: agents add capability and add risk.

**Strong answer:**

"It depends on the query mix. **It improves things when questions need more than one retrieval, or a non-document source.** It **hurts** latency, cost and predictability for simple lookups.

**Where agents help:**
- **Multi-source:** 'What's my remaining leave, and what's the carry-forward policy?' That needs an HRMS API (user data) plus the policy docs (RAG).
- **Computation:** aggregations over tables → a SQL or Python tool instead of making the LLM do arithmetic.
- **Multi-hop and decomposition:** the agent issues sub-queries and checks sufficiency.
- **Corrective behavior:** if retrieval is weak, rewrite and retry, or ask the user a clarifying question.
- **Actions:** after answering, 'raise a leave request', which is tool execution with approval.

**Where they hurt:**
- Simple FAQ lookups: an agent loop turns a 1.5 s answer into 5 s at 3–5x the cost.
- Reliability: more LLM decisions means more ways to fail (wrong tool, loops).
- Evaluation becomes harder (trajectories, not just answers).

**How I'd integrate it:**
```
Query -> Router (rules/small classifier)
          |- simple factual  -> standard RAG pipeline (fast path, ~70-80% traffic)
          |- personal data   -> API tool + RAG
          |- analytical      -> SQL tool
          '- complex/multi   -> agentic RAG with caps
```
Tools have typed schemas, permissions scoped to the user's identity, and read-only by default. Write actions need confirmation.

I'd prove the value with a per-segment eval. For example, the agentic path lifted accuracy on multi-part questions from 58% to 81%, while the fast path kept simple-query P50 at 1.4 s."

**Follow-ups:**
- *How do you evaluate the router?* → A labeled route set; measure misroutes, especially complex queries sent to the fast path.

**Red flags:**
- "Agents always make RAG better."
- Sending all traffic through an agent loop.

---

<a id="q11"></a>
## Q11. With 1M-token context windows, do we still need RAG?
*[Added]*

**What the interviewer is testing:** a nuanced view of long context vs retrieval, grounded in economics.

**Strong answer:**

"For many cases, yes, but the boundary is moving.

**Long context wins when:**
- The corpus is small and bounded (one contract, a codebase module, a few reports), so you can put all of it in the window.
- The task needs a **holistic** read (summarize or compare whole documents) rather than a lookup.
- You can reuse the context with **prompt caching** across many questions on the same document.

**RAG still wins when:**
- **Scale:** enterprise corpora are GBs to TBs, which is millions of tokens times thousands. It doesn't fit.
- **Cost:** 500k input tokens per query, even cached, costs far more than 5k retrieved tokens. At 100k queries a day, that's the difference between a viable product and not.
- **Latency:** prefilling hundreds of thousands of tokens adds seconds to time to first token.
- **Access control:** RAG filters documents per user. Stuffing everything into the context risks leakage.
- **Freshness and citations:** retrieval gives explicit provenance and easy updates.
- **Accuracy:** effective recall across very long contexts still degrades on many models with realistic distractors (see the lost-in-the-middle tests).

**What I'd actually do:** a hybrid. Use retrieval to select the relevant *documents* (not tiny chunks), then use a long context to read them whole. That shifts chunking concerns towards document-level retrieval."

**Follow-ups:**
- *How would you decide for a specific product?* → Measure accuracy, cost and latency for both on the golden set at expected QPS.

**Red flags:**
- "RAG is dead."
- "Long context doesn't work", with no nuance.

---

<a id="q12"></a>
## Q12. Parent-document / small-to-big / sentence-window retrieval: when to use them?
*[Added]*

**What the interviewer is testing:** a practical fix for the chunk-size dilemma.

**Strong answer:**

"These techniques decouple the **retrieval unit** from the **context unit**. Small chunks embed precisely; big chunks give the LLM enough context.

- **Small-to-big / parent-document:** index small child chunks (for example 100–200 tokens). On a hit, return the parent section (for example 1,000–2,000 tokens) to the LLM. Store `parent_id` on each child and dedupe parents when several children hit.
- **Sentence-window:** index single sentences. On a hit, return ±k surrounding sentences (for example k = 3).
- **Hierarchical / auto-merging:** if many children of the same parent are retrieved, merge them up into the parent.

**When to use them:**
- Answers need surrounding context (policies with conditions and exceptions, legal clauses, technical procedures).
- Plain chunk-size tuning is stuck: small chunks give incomplete answers, large chunks give diluted retrieval.

**Trade-offs:**
- More tokens per retrieved hit, so cap the number of parents (for example 3).
- More storage and bookkeeping: child → parent mapping, and a document store for the parents.
- If parents are huge, you've recreated the large-chunk problem at the context level.

Example: on a policy corpus, 150-token children with 1,200-token parents improved answer completeness (judge-scored) from 3.4 to 4.1 out of 5, at about 1.8x input tokens."

**Follow-ups:**
- *Interaction with the reranker?* → Rerank at the child level (precise), then expand to parents. Or rerank parents if they fit the reranker's max length.

**Red flags:**
- Not knowing that the retrieval unit and the generation unit can differ.

---

## Rapid-fire recap

- Choose advanced techniques by the failure mode they fix, and add them in order of cost.
- RAG Fusion's main latency is the query-generation call. Retrievals run in parallel; make it adaptive.
- CRAG grader: reranker-score thresholds first, then a small batched grader, and rarely an expensive fallback.
- CRAG corrects retrieval with bounded cost; Self-RAG critiques generation with variable cost and needs a specialized model.
- Contextual Retrieval: prepend LLM-generated document context to each chunk for both embeddings and BM25. Prompt caching and batch APIs make it affordable at scale.
- GraphRAG is for multi-hop and global questions over relationships. Indexing is expensive and harder to keep fresh.
- Multi-hop stops on a sufficiency check or no new information, with hard caps on hops, tokens and time.
- A/B test HyDE offline by query segment first, then online randomized by user, with latency guardrails.
- Agentic RAG limits come from trace distributions (P95 of successful runs), enforced in code, with graceful stops.
- Route queries: fast RAG path for simple lookups, agentic path only for complex ones.
- Long context complements RAG; it doesn't replace it at enterprise scale, cost or ACL requirements.
- Small-to-big: retrieve precise children, give the LLM the parent context.
