# 04 — RAG Core Pipeline

**What interviewers probe here:** whether you understand RAG as a *search system plus a generation step*, not "vector DB + LLM". Expect scenario questions on chunking, hybrid retrieval, fusion, reranking, document parsing and citations, and expect to be asked *why* you'd pick each component and *what breaks* without it.

[← Back to index](README.md)

## Questions

1. [Explain an end-to-end RAG pipeline](#q1)
2. [Why RAG is not just "vector DB + LLM"](#q2)
3. [Scenario-based chunking and chunk size](#q3)
4. [How much chunk overlap is enough?](#q4)
5. [Choosing Top-K](#q5)
6. [Dense vs sparse; semantic vs keyword vs hybrid](#q6)
7. [How BM25 is calculated](#q7)
8. [Combining BM25 and semantic results: RRF](#q8)
9. [Why a reranker after vector search?](#q9)
10. [How the system decides which document is relevant](#q10)
11. [The LLM is getting irrelevant context](#q11)
12. [Testing "lost in the middle"](#q12)
13. [Designing citations and source grounding](#q13)
14. [Large tables spanning multiple pages](#q14)
15. [Scanned PDFs, images, charts and multimodal extraction](#q15)
16. [RAG vs relying on model knowledge](#q16)
17. [Query understanding: rewriting, expansion, multi-query](#q17)
18. [Conversational follow-ups in RAG](#q18)

---

<a id="q1"></a>
## Q1. Explain an end-to-end RAG pipeline (indexing phase vs retrieval + generation phase) and every component.
*Source: interview posts*

**What the interviewer is testing:** whether you can describe the whole system, separate offline from online work, and say what each stage is responsible for.

**Strong answer:**

"I think of RAG as two pipelines that share a data store.

**Offline indexing** (batch or streaming):
```
Sources (PDF, Confluence, DB, APIs)
  -> Load & parse (text, tables, images, layout)
  -> Clean & normalize (dedupe, strip boilerplate, extract metadata)
  -> Chunk (structure-aware)
  -> Embed (dense) + index tokens (BM25/sparse)
  -> Store: vector index + keyword index + metadata + raw chunk text
```

**Online retrieval + generation** (per request, latency-sensitive):
```
User query
  -> Query understanding (rewrite with chat history, classify intent, extract filters)
  -> Retrieve: dense + BM25 in parallel, with metadata/ACL filters
  -> Fuse (RRF) -> top ~50
  -> Rerank (cross-encoder) -> top 5-8
  -> Build context (order, dedupe, attach source ids, fit token budget)
  -> LLM with a grounding prompt ("answer only from context, cite, else say you don't know")
  -> Post-process: citation check, guardrails, format
  -> Response + trace logged for eval
```

Each stage has a different job. Parsing and chunking decide **what can be found at all**. Retrieval optimizes **recall**. Reranking optimizes **precision**. Context building handles the **token budget**. The prompt enforces **faithfulness**. The eval loop tells me which stage broke.

For an HR-policy assistant I built, about 12k documents gave roughly 180k chunks. Hybrid retrieval plus a reranker took answer accuracy on our 300-question golden set from 71% to 86%, and P95 latency was about 2.4 s, most of it LLM generation."

**Follow-ups:**
- *Where do you store raw text: in the vector DB or separately?* → Either works. I keep the chunk text and metadata with the vector for simplicity, but the source of truth is a document store with versions, so I can re-embed without re-parsing.
- *What's the first thing you'd instrument?* → Retrieval recall@K on a golden set, because a generation fix can't recover a chunk that was never retrieved.
- *Sync or async indexing?* → Async, through a queue, so ingestion spikes never affect query latency.

**Red flags:**
- Describing only "embed, store, similarity search, LLM", with no parsing, reranking, filtering or eval.
- Not separating offline and online concerns.
- Having no answer for how they know the pipeline works.

---

<a id="q2"></a>
## Q2. Why is RAG not just "vector DB + LLM"?
*Source: interview posts*

**What the interviewer is testing:** production maturity, and whether you know where real failures come from.

**Strong answer:**

"'Vector DB + LLM' is the demo. In production, most failures happen in the layers around those two:

| Layer | What goes wrong if you skip it |
|---|---|
| Parsing | Tables become word soup; scanned PDFs are empty |
| Chunking | Answers are split across chunks, or chunks mix topics |
| Hybrid search | Exact terms (SKUs, error codes, names) are missed by embeddings |
| Metadata/ACL filters | Wrong version, wrong region, or data leaks across users |
| Reranking | The right chunk sits at rank 14 and never reaches the model |
| Context building | Duplicates waste tokens; the key fact gets lost in the middle |
| Grounding prompt + refusal | The model fills gaps from its priors, which is hallucination |
| Freshness/re-indexing | Stale prices and deleted policies still get answered |
| Evaluation & tracing | You can't tell whether a change helped |

So RAG is search engineering plus prompt engineering plus data engineering plus evaluation. I've seen a case where retrieval was perfect but the prompt was built before retrieval returned, because of a fire-and-forget async call. The model answered with zero context. No embedding or model change would have fixed that. The fix was in the plumbing."

**Follow-ups:**
- *Which of those layers gives the biggest win first?* → Usually parsing and chunking quality, then hybrid search plus a reranker.
- *Can you skip the vector DB entirely?* → Yes, for small corpora (BM25 alone, or long context) or for structured data (SQL). The vector DB is an implementation detail.

**Red flags:**
- "Just use a better model."
- Not knowing that retrieval quality caps answer quality.

---

<a id="q3"></a>
## Q3. Scenario-based chunking: which strategy for unstructured text, structured data, and PDFs with images, tables and text? How does chunk size affect retrieval?
*Source: interview posts*

**What the interviewer is testing:** whether you pick a chunking strategy based on document structure and query type instead of a default like "512 tokens".

**Strong answer:**

"Chunking should follow the **natural unit of meaning** in the document and the **granularity of the questions** users ask.

| Content | Strategy | Why |
|---|---|---|
| Prose (articles, emails) | Recursive splitting (paragraph → sentence → token) with a max size | Keeps paragraphs intact and degrades gracefully |
| Topic-shifting prose (transcripts) | Semantic chunking: split where the embedding similarity between sentences drops | Boundaries follow topic changes |
| Markdown/HTML/docs with headings | Structure-aware: split by heading, prepend the heading path ("Leave Policy > Maternity > Eligibility") | The heading carries context the paragraph lacks |
| Structured data (product catalog, CRM rows) | Entity-based: one chunk per entity, with fields serialized as key: value | Queries are about one entity |
| Code | AST-based: split by function or class | Keeps semantic units whole |
| PDF with text, tables and images | Layout-aware parsing (Unstructured, Docling, Azure Document Intelligence, LlamaParse): separate text blocks, tables and figures, then chunk each type differently | A fixed-size split cuts tables in half |
| FAQs | One Q&A pair per chunk | Perfect retrieval unit |

**How chunk size affects retrieval:**
- **Too small** (for example 100 tokens): precise embeddings but missing context. "It is 26 weeks" with no subject. Recall looks fine, but answers are incomplete.
- **Too large** (for example 2,000 tokens): the embedding averages several topics, so it gets diluted and matches weakly. You also waste context tokens.
- A sweet spot of **300–800 tokens** is common for prose, but I tune it on a golden set: I try 256, 512 and 1024, measure recall@5 and answer accuracy, and choose the best.

A pattern I like is **small-to-big**: embed small chunks for precise matching, then return the parent section to the LLM for context.

For example, on an insurance-policy corpus, fixed 512-token chunks gave recall@5 of 0.68. Heading-aware chunks with the heading path prepended reached 0.81, with no other change."

**Follow-ups:**
- *Does chunk size depend on the embedding model?* → Yes. Stay well under the model's max input (many truncate at 512 tokens), because text past that limit is silently ignored.
- *Semantic chunking always better?* → No. It costs more to compute and is sometimes no better than recursive splitting plus headings. Measure it.
- *How do you pick for mixed corpora?* → Route by document type at ingestion and use a different chunker per type.

**Red flags:**
- "I always use 1000 characters with 200 overlap" without saying why.
- Not knowing embedding models truncate input.
- Treating tables like prose.

---

<a id="q4"></a>
## Q4. How much chunk overlap is enough?
*Source: interview posts*

**What the interviewer is testing:** whether you know what overlap is actually for, and what it costs.

**Strong answer:**

"Overlap is insurance against a fact straddling a boundary. It isn't free. It inflates index size, and it creates near-duplicate chunks that crowd the top-K.

My defaults:
- **Fixed-size chunking:** 10–20% overlap, for example 50–100 tokens on a 512-token chunk. That's usually enough to carry a sentence or two across the boundary.
- **Structure-aware or semantic chunking:** little or no overlap, because boundaries already fall at natural breaks. Instead, I prepend context such as the heading path or document title.
- **Sentence-window retrieval:** no overlap in the index. I expand by ±N sentences at retrieval time.

How I'd decide empirically: build a golden set where some answers span chunk boundaries, then sweep overlap at 0%, 10%, 20% and 30%. Plot recall@K against index size. Typically recall plateaus around 10–15% and more overlap only adds duplicates.

A side effect to handle: with overlap, two adjacent chunks often both land in the top 5. I **dedupe or merge adjacent chunks** during context building so I don't pay tokens twice."

**Follow-ups:**
- *Overlap at the character or token level?* → Token or sentence level, so words and sentences aren't cut mid-way.
- *What's the storage cost of 20% overlap?* → Roughly 20–25% more chunks, which means more vectors and more embedding cost.

**Red flags:**
- "More overlap is always safer."
- Not mentioning duplicate chunks in the top-K.

---

<a id="q5"></a>
## Q5. How do you choose Top-K?
*Source: interview posts*

**What the interviewer is testing:** understanding of the recall/precision/cost trade-off across the retrieval stages.

**Strong answer:**

"I choose K **per stage**, because the stages have different goals:

```
Retriever:  K = 50-100   (maximize recall, cheap)
Reranker:   keep 5-10    (maximize precision)
LLM context: 3-8 chunks  (fit the token budget, minimize noise)
```

How I tune it:
1. On a golden set, plot **recall@K** for the retriever at K = 5, 10, 20, 50 and 100. Pick the K where the curve flattens, for example recall@50 = 0.94 and recall@100 = 0.95, so K = 50.
2. For the final context, sweep 3, 5, 8 and 12 and measure **answer accuracy and faithfulness**. Too few chunks misses multi-part answers. Too many adds noise and lost-in-the-middle effects, and costs more.
3. Consider a **dynamic K**: cut off by a reranker score threshold instead of a fixed number. A simple lookup might need one chunk; a comparison question might need eight.

The cost side: each extra 500-token chunk at 1M queries a month is 500M input tokens. That's real money, so I don't pad the context 'just in case'."

**Follow-ups:**
- *Why not just pass 50 chunks to a long-context model?* → Higher cost, higher latency, and accuracy often drops because of noise and position effects.
- *Different K per query type?* → Yes. Route "compare X and Y" queries to a higher K, or run a sub-query per entity.

**Red flags:**
- A single hardcoded K = 3 with no reasoning.
- Not distinguishing retriever K from context K.

---

<a id="q6"></a>
## Q6. Dense vs sparse retrieval; semantic vs keyword vs hybrid. When can keyword search beat hybrid?
*Source: interview posts*

**What the interviewer is testing:** whether you know the strengths of each retrieval family and choose based on the query distribution.

**Strong answer:**

"**Sparse retrieval** (BM25, TF-IDF, SPLADE) represents text as weighted terms. It excels at **exact matches**: product codes, error messages, names, acronyms and rare terms. It's cheap, explainable and needs no training.

**Dense retrieval** (bi-encoder embeddings) represents meaning as a vector. It excels at **paraphrase and semantics**: 'time off after having a baby' matches 'maternity leave'. It's weak on rare tokens and out-of-domain jargon.

**Hybrid** runs both and fuses the results, usually with RRF. It's the safest default for mixed enterprise queries, typically adding 5–15 points of recall over either one alone.

| Query type | Best |
|---|---|
| "error E4021 on login" | Keyword |
| "why can't users sign in after the update" | Dense |
| Mixed enterprise search | Hybrid |

**When keyword can beat hybrid:**
- Queries are dominated by **identifiers or exact phrases** (legal citations, part numbers, log search). Dense results add noise, and fusion can push the exact match down.
- **Out-of-domain vocabulary** the embedding model never saw (internal codenames, chemistry, a non-English domain), so dense scores are close to random.
- **Very short documents or titles**, where BM25 is already near perfect.
- A badly weighted fusion: if dense recall is poor, giving it equal weight in RRF hurts.

That's why I don't assume hybrid is best. I segment the golden set by query type and measure each retriever. Sometimes the right answer is a **router**: if the query contains an ID pattern, go to keyword-first."

**Follow-ups:**
- *What is SPLADE?* → A learned sparse model. It expands terms with learned weights, which gives semantic matching in an inverted index.
- *How do you weight dense vs sparse?* → Weighted RRF or a convex combination of normalized scores, tuned on the golden set.

**Red flags:**
- "Semantic search is always better than keyword."
- Not knowing BM25 is still a strong baseline.

---

<a id="q7"></a>
## Q7. How is the BM25 score calculated?
*Source: interview posts*

**What the interviewer is testing:** whether you actually know the formula and the intuition behind each term.

**Strong answer:**

"BM25 scores a document D for a query Q by summing over the query terms:

```
score(D, Q) = Σ_{q in Q}  IDF(q) · [ f(q,D) · (k1 + 1) ] / [ f(q,D) + k1 · (1 − b + b · |D| / avgdl) ]

IDF(q) = ln( (N − n(q) + 0.5) / (n(q) + 0.5) + 1 )
```

- `f(q,D)` is the term frequency of q in D.
- `|D|` is the document length and `avgdl` is the average document length.
- `N` is the number of documents and `n(q)` is the number of documents containing q.
- `k1` (usually 1.2–2.0) controls **term-frequency saturation**. The 10th occurrence of a word adds much less than the 1st.
- `b` (usually 0.75) controls **length normalization**. Long documents don't win just by containing more words.

The intuition is three ideas: rare terms matter more (IDF), repeating a term has diminishing returns (saturation with k1), and a match in a short document means more than one in a long document (b).

Compared with TF-IDF, the saturation and length normalization are what make BM25 robust.

In practice I'd tune it: for short chunks of similar length, lowering b matters less; for highly repetitive technical docs, I'd lower k1."

**Follow-ups:**
- *Does BM25 handle synonyms?* → No. That's why we pair it with dense retrieval or add query expansion.
- *Tokenization matters?* → A lot: stemming, lowercasing and handling of codes like "E-4021". Bad tokenization breaks exact matching.

**Red flags:**
- Only saying "it's like TF-IDF" with no idea of saturation or length normalization.

---

<a id="q8"></a>
## Q8. BM25 returns n chunks and semantic search returns n chunks. How do you combine them? Explain Reciprocal Rank Fusion.
*Source: interview posts*

**What the interviewer is testing:** whether you know that raw scores from different retrievers aren't comparable, and how rank fusion solves that.

**Strong answer:**

"You can't just add the scores. BM25 scores are unbounded (for example 3–25) and cosine similarity sits in a narrow band (for example 0.7–0.9). The scales are incomparable. There are two options:

**1. Reciprocal Rank Fusion (RRF)** uses ranks and ignores scores:
```
RRF(d) = Σ_{r in retrievers}  1 / (k + rank_r(d))      with k = 60 by convention
```
A document ranked 1st in BM25 and 3rd in dense gets 1/61 + 1/63. A document found by only one retriever gets a single term. So documents that **both** retrievers like rise to the top. k = 60 dampens the advantage of the very top ranks, so one retriever's #1 doesn't dominate.

```python
def rrf(result_lists, k=60, top_n=20):
    scores = {}
    for results in result_lists:          # each is a list of doc_ids ordered by rank
        for rank, doc_id in enumerate(results, start=1):
            scores[doc_id] = scores.get(doc_id, 0.0) + 1.0 / (k + rank)
    return sorted(scores, key=scores.get, reverse=True)[:top_n]
```

**2. Score normalization plus a weighted sum**: min-max or z-score normalize each list, then `α·dense + (1−α)·sparse`. This keeps score magnitudes, but it's sensitive to outliers and needs α tuned per corpus.

I default to RRF because it's parameter-light and robust. If I have a good golden set I try **weighted RRF** (a weight per retriever) or tuned α. Either way, the fused top ~50 goes to the reranker, which does the precise ordering.

Example: on a support-ticket corpus, dense alone had recall@20 of 0.78, BM25 alone 0.74, and RRF 0.89."

**Follow-ups:**
- *Why k = 60?* → It's the empirical value from the original RRF paper. It works across many settings. Smaller k rewards the top ranks more.
- *What if a doc appears in only one list?* → It still scores, but lower. That's desirable, because it keeps recall.
- *Do vector DBs support this natively?* → Yes. Weaviate, Qdrant, Elasticsearch/OpenSearch and others offer hybrid search with RRF or relative-score fusion.

**Red flags:**
- Summing raw BM25 and cosine scores.
- Concatenating both lists and taking the first n.

---

<a id="q9"></a>
## Q9. If vector search already returns the top 10, why add a reranker (e.g. BGE)? When should you add one?
*Source: interview posts*

**What the interviewer is testing:** whether you understand bi-encoders vs cross-encoders, and a two-stage retrieval design.

**Strong answer:**

"The first-stage retriever is a **bi-encoder**. The query and the document are embedded **separately**, and similarity is one dot product. That's what makes it fast over millions of vectors. But it compresses a whole chunk into one vector before it ever sees the query, so it can't model fine-grained interaction such as negation, exact constraints, or which entity the question is about.

A **reranker** (cross-encoder, such as BGE-reranker, Cohere Rerank or ms-marco MiniLM) takes **query and document together** through the transformer, so every query token attends to every document token. The relevance judgment is much more precise, but you need one forward pass per (query, doc) pair, so you can only afford it on tens of candidates.

```
Millions of chunks --bi-encoder ANN (~20 ms)--> top 50 (recall)
top 50 --cross-encoder (~50-150 ms on GPU)--> top 5 (precision)
```

So stage one is optimized for **recall at low cost** and stage two for **precision**. They're complementary.

**When I add a reranker:**
- Recall@50 is high but precision@5 is low: the right chunk is retrieved but sits at rank 15.
- After hybrid fusion, since RRF ranks are rough.
- When many chunks are near duplicates or on-topic but not answering (versioned docs, policies).

**When I might not:** a very tight latency budget (a voice agent), a tiny corpus, or an eval showing no gain.

Example: a BGE reranker on the top 40 moved MRR from 0.61 to 0.79 and answer accuracy up 9 points, for about 80 ms extra at P50."

**Follow-ups:**
- *Can an LLM be the reranker?* → Yes (listwise or pointwise LLM reranking). It's more accurate on complex relevance but slower and more expensive. I'd use it for low-QPS, high-value queries.
- *How many candidates do you rerank?* → 20–100. Beyond that the latency grows linearly and the gains flatten.
- *Reranker score as a refusal threshold?* → Yes. If the best score is below a calibrated threshold, answer "I don't have that information".

**Red flags:**
- "The reranker just sorts by similarity again."
- Not knowing the bi-encoder vs cross-encoder distinction.

---

<a id="q10"></a>
## Q10. What roles do embeddings, similarity scores, metadata filters and reranking play in deciding which document is relevant?
*Source: interview posts*

**What the interviewer is testing:** whether you can explain "how the system chooses" as a sequence of narrowing filters.

**Strong answer:**

"Relevance is decided in layers, each narrowing the candidate set:

1. **Metadata filters: 'which documents am I allowed and supposed to look at?'** Tenant, ACL, document type, product, region, `is_latest_version=true`, date ranges. These are hard constraints and should run **before or during** the ANN search, not after.
2. **Embeddings + similarity: 'what is semantically close?'** The query embedding is compared with chunk embeddings by cosine or dot product. The score is **relative, not absolute**: 0.82 doesn't mean '82% relevant', and thresholds vary by model.
3. **Keyword scores (BM25): 'what matches exactly?'** They catch identifiers that embeddings blur.
4. **Fusion (RRF)** merges the two views.
5. **Reranking: 'which of these actually answers this question?'** A cross-encoder judges query and chunk together.
6. **LLM: 'what to use from the context.'** With a grounding prompt it picks the relevant pieces and cites them, and it should say so if nothing answers the question.

Example: 'What's the WFH policy for the Pune office?' The filter is `location=Pune, doc_type=policy, latest=true`. Dense search finds 'remote work guidelines'. BM25 catches 'WFH'. The reranker promotes the chunk that actually states the number of days."

**Follow-ups:**
- *Where do filters come from?* → The user's auth context (ACL, tenant), plus fields extracted from the query by an LLM or a rules-based parser.
- *Similarity threshold for "no answer"?* → Better to threshold on the reranker score, which is more calibrated, and validate it on a set of unanswerable questions.

**Red flags:**
- Treating the cosine score as a probability.
- Applying security filters after retrieval, which leaks data or empties the results.

---

<a id="q11"></a>
## Q11. The LLM is getting irrelevant context. How do you improve retrieval?
*Source: interview posts*

**What the interviewer is testing:** a structured debugging method rather than random tweaks.

**Strong answer:**

"First I'd **measure before changing anything**: take 50–100 failing queries, label which chunks *should* have been retrieved, and compute recall@K and precision@K. Then I'd work through the pipeline in order.

1. **Is the answer even in the index?** Check parsing (tables, scanned pages) and chunking (answer split in half). This is surprisingly often the problem.
2. **Query side:** vague or conversational queries ('what about that one?') → rewrite them using history, expand them, or use multi-query. For jargon, add a synonym map.
3. **Retriever:** add BM25 for hybrid search if exact terms are being missed; try a better or domain-tuned embedding model; check that the query uses the same model version and prefix (e.g. `query:` / `passage:` for E5/BGE).
4. **Filters:** add metadata filters (product, version, region) so the search space is correct. Many 'irrelevant' chunks come from the wrong version or product.
5. **Reranker:** if recall is fine but precision is poor, add or upgrade the cross-encoder and apply a score cutoff.
6. **Context building:** dedupe, drop chunks below a threshold, order by relevance, cap the count.
7. **Chunk enrichment:** prepend titles and heading paths, or use contextual retrieval, so each chunk carries its own context.

I'd change one thing at a time and re-run the eval set, keeping a results table per experiment."

**Follow-ups:**
- *Precision is low but recall is fine. What first?* → Reranker plus a score threshold.
- *Recall is low. What first?* → Parsing and chunking checks, then hybrid search, then query rewriting.

**Red flags:**
- "Increase top-K" as the only answer (that makes noise worse).
- Changing several things at once with no eval.

---

<a id="q12"></a>
## Q12. How do you test for "lost in the middle"?
*Source: interview posts*

**What the interviewer is testing:** knowledge of position bias in long contexts and how to measure it empirically.

**Strong answer:**

"'Lost in the middle' (Liu et al., 2023) is the finding that LLMs use information at the **start and end** of the context better than information in the **middle**. The result is a U-shaped accuracy curve.

**How I'd test it on my own system:**
1. Take a set of questions with a known gold chunk.
2. Build contexts of N chunks (for example 5, 10 and 20): one gold chunk plus distractors that are realistic retrieved chunks, not random ones.
3. Place the gold chunk at each position 1..N and run each configuration several times.
4. Measure answer accuracy by position. Plot accuracy against position for each N.
5. Repeat for each model I'm considering, since the effect varies a lot by model and has shrunk in newer models.

A needle-in-a-haystack test is a simpler variant, but it's easier than real RAG because a synthetic needle stands out. Realistic distractors are what matter.

**Mitigations if the effect shows up:**
- Pass fewer, better chunks (reranker plus a cutoff). This is the biggest lever.
- Put the highest-ranked chunks at the start and end (a 'sandwich' ordering), or at the end, just before the question.
- Repeat the question after the context.
- Split multi-part questions into sub-queries."

**Follow-ups:**
- *Does long context fix it?* → No. A bigger window doesn't guarantee uniform attention. Test it.
- *How many runs?* → Enough for confidence intervals, for example 100+ questions per position.

**Red flags:**
- Having heard the term but having no test methodology.
- Testing with random distractors only.

---

<a id="q13"></a>
## Q13. Design citations and source grounding down to document, page, paragraph and line.
*Source: interview posts*

**What the interviewer is testing:** end-to-end design, from ingestion metadata to UI verification, and whether you make citations trustworthy rather than decorative.

**Strong answer:**

"Citations have to be designed **at ingestion time**. You can't recover page numbers later.

**1. Ingestion: keep provenance on every chunk:**
```json
{ "chunk_id": "doc42_p7_c3", "doc_id": "doc42", "doc_version": "v5",
  "title": "Travel Policy", "page": 7, "section": "4.2 Per diem",
  "char_start": 1840, "char_end": 2512, "bbox": [72, 310, 540, 460],
  "url": "https://.../doc42.pdf#page=7" }
```
Layout-aware parsers give page numbers and bounding boxes. For HTML I keep anchors.

**2. Prompting:** give each chunk a short id in the context (`[S1]`, `[S2]`) and instruct: 'Every claim must cite one or more source ids. If no source supports a claim, don't make it.' Structured output: `{answer, citations: [{source_id, quote}]}`.

**3. Verification (the part most people skip):**
- Check that every cited id was actually in the context.
- Check that the quoted span exists **verbatim or by fuzzy match** in that chunk. This catches fabricated quotes.
- Optionally, an NLI or LLM check that the sentence is entailed by the cited chunk.
- Drop or flag claims that fail.

**4. UI:** clickable citations open the PDF at the page and highlight the bounding box or character span, so the user can verify in one click.

**Trade-offs:** the verification step adds 100–300 ms; sentence-level citations are more precise but noisier. For legal or medical use, I'd require quote-level citations and block unsupported sentences."

**Follow-ups:**
- *Model cites S3 but the claim is from S1?* → The quote-match check catches it. Remap to the chunk that actually contains the quote.
- *Provider-native citation features?* → Some APIs (for example Anthropic's citations on documents) return cited spans natively, which removes the parsing step. I'd still verify.

**Red flags:**
- "The LLM will just say which document it used."
- No provenance metadata stored at ingestion.

---

<a id="q14"></a>
## Q14. How do you handle large tables that span multiple pages? What are effective table parsing and ingestion strategies?
*Source: interview posts*

**What the interviewer is testing:** practical document-engineering experience, which is one of the most common real-world RAG failure points.

**Strong answer:**

"Tables break naive RAG in two ways: text extraction flattens them into word soup, and fixed-size chunking cuts them mid-row, so the header gets separated from the values.

**Parsing:**
- Use a table-aware parser: Docling, Unstructured (hi_res), Azure Document Intelligence, AWS Textract, Camelot/pdfplumber for digital PDFs, or a vision LLM for messy scans.
- **Multi-page tables:** detect continuation (same column count and widths, no new header, or a 'continued' marker, on the next page) and **stitch** the pages into one logical table. Carry the header row forward if it's repeated or missing on later pages.

**Representation, choosing per table:**
1. **Row-level chunks with the header repeated:** each row, or group of ~10–20 rows, is serialized as `Column: value` pairs plus the table title and caption. This is great for lookups ('premium for plan B, age 40').
2. **Table summary chunk:** an LLM-generated description ('Table 3 lists premiums by plan and age band for 2025'). It's embedded for discovery and points to the full table.
3. **Structured storage:** load the table into SQL or a dataframe, and have the agent query it (text-to-SQL or pandas) for aggregations ('average premium across plans'). Retrieval is bad at arithmetic.
4. Keep a markdown/HTML version for the LLM context, since models read markdown tables well.

**Ingestion pipeline:**
```
PDF -> layout parse -> detect tables -> stitch cross-page -> normalize headers
    -> (a) row-group chunks w/ headers  (b) summary chunk  (c) SQL table
    -> all linked by table_id + page range
```

In a pricing-document RAG I worked on, moving from plain text extraction to row chunks with repeated headers took table-question accuracy from about 40% to 85%."

**Follow-ups:**
- *Merged cells or nested headers?* → Flatten to a composite header ("2025 > Q1 > Revenue").
- *How do you validate table extraction?* → Sample checks against the source, row and column count checks, and a small golden set of table questions.

**Red flags:**
- "PyPDF extracts the text, then chunk it."
- Not recognizing that aggregation questions need SQL, not retrieval.

---

<a id="q15"></a>
## Q15. How do you extract text from scanned PDFs, images, charts and other formats, including with multimodal models?
*Source: interview posts*

**What the interviewer is testing:** knowledge of OCR vs layout models vs vision-LLMs, with their cost and accuracy trade-offs.

**Strong answer:**

"First, **detect the document type**. A PDF with no text layer, or with garbage characters, is a scan and needs OCR. Then choose by quality needs and volume:

| Approach | Good for | Trade-off |
|---|---|---|
| Classic OCR (Tesseract, PaddleOCR) | Clean scans, high volume | Cheap; poor on layout and handwriting |
| Cloud document AI (Azure Document Intelligence, Textract, Google Document AI) | Forms, tables, key-value pairs, layout | Good accuracy, per-page cost, data residency |
| Layout-aware open parsers (Docling, Unstructured, Marker) | Mixed PDFs with reading order and tables | Self-hosted, needs a GPU for best results |
| Vision LLMs (GPT-4o/Claude/Gemini-class, Qwen-VL) | Charts, diagrams, handwriting, messy layouts | Best understanding; expensive and slower; can hallucinate numbers |

**Charts and images:** use a vision LLM to generate a **structured description** ("Bar chart: revenue by quarter 2024, Q1 = 1.2M, ..."), then store the description as a chunk with a link to the image. For key figures, keep the image itself so that a multimodal model can see it at answer time.

**Multimodal embeddings** (CLIP-style, or ColPali-style page-image retrieval) are an option. You embed page images directly and skip OCR. This is powerful for visually rich docs, at the cost of storage (multi-vector) and needing a vision model to answer.

**Pipeline I'd use:** route per page (digital text → direct extraction; scan → OCR/layout model; figure → vision LLM caption). Keep a confidence score, and send low-confidence pages to a stronger model or to human review.

Cost check: 1M pages through a vision LLM at about $0.005–0.01 a page is $5k–10k, so I use it only where cheaper tools fail."

**Follow-ups:**
- *Vision LLM invents a number from a chart?* → Cross-check against OCR text if present, keep the image link for verification, and flag the chunk as `source=vlm_description`.
- *Other formats?* → DOCX/PPTX/HTML have native structure: parse the structure, not a PDF render. Email: strip signatures and quoted threads.

**Red flags:**
- "Just run everything through GPT-4 Vision" with no thought about cost or accuracy.
- Not detecting scanned vs digital PDFs.

---

<a id="q16"></a>
## Q16. When should the system use RAG instead of relying on model knowledge?
*Source: interview posts*

**What the interviewer is testing:** judgment about when retrieval adds value and when it only adds latency.

**Strong answer:**

"Use RAG when the answer depends on knowledge that is:
- **Private** (internal docs, customer data): the model never saw it.
- **Fresh or changing** (prices, policies, inventory, news): after the training cutoff, or changing weekly.
- **Needing attribution** (legal, medical, compliance): users need a citation.
- **Precise and long-tail** (exact numbers, clauses, IDs): models are unreliable on rare facts.
- **Per-tenant** (each customer has different policies).

Rely on the model's own knowledge for general reasoning, language tasks (rewrite, summarize, translate), common knowledge, and coding patterns. Retrieval there adds latency and noise.

In an agentic setup I make it a **decision**: retrieval is a tool, and a lightweight router (rules, a small classifier, or the LLM's tool choice) decides. For example, 'Rephrase this email' needs no retrieval; 'What's our refund window for enterprise plans?' must retrieve and must refuse if nothing is found.

The trade-off is that always retrieving is simpler and safer for factual domains. Skipping retrieval saves 100–300 ms and tokens on chit-chat. For a customer-facing bot about company facts, I default to always retrieving plus a strict grounding prompt, because the cost of a hallucinated price outweighs the latency saved."

**Follow-ups:**
- *How do you evaluate the router?* → Labeled set of queries marked needs-retrieval yes/no; track false negatives (skipped retrieval when needed), which are the dangerous ones.
- *What about fine-tuning instead?* → Fine-tune for behavior and format; retrieve for facts (see 14-fine-tuning).

**Red flags:**
- "The model knows most things, so RAG is only for PDFs."
- Not considering freshness or tenancy.

---

<a id="q17"></a>
## Q17. Query understanding: query rewriting, expansion and multi-query.
*Source: interview posts*

**What the interviewer is testing:** whether you know that user queries are often poor search queries, and how to fix them cheaply.

**Strong answer:**

"Users write queries for humans, not retrievers. They're short, vague, conversational and full of typos. Query understanding sits before retrieval:

1. **Rewriting / condensation:** turn a follow-up plus chat history into a standalone query ("and for contractors?" → "What is the leave policy for contractors?"). Use a small, fast LLM here.
2. **Expansion:** add synonyms and acronyms ("WFH" → "work from home, remote work"), from a domain synonym table or an LLM. This mainly helps BM25.
3. **Multi-query:** generate 3–5 paraphrases or perspectives, retrieve for each, and fuse with RRF. It raises recall on ambiguous queries at the cost of more retrieval calls.
4. **Decomposition:** split compound questions ("compare plan A and B premiums") into sub-queries, one per entity.
5. **Intent classification and filter extraction:** detect doc type, product or date ("2024 policy") and turn it into metadata filters.
6. **HyDE:** generate a hypothetical answer and embed that. It helps when queries and documents look very different (see file 05).

Cost and latency: each LLM step adds 100–400 ms. I run the rewrite on a small model, run the sub-queries in parallel, and cache rewrites for frequent queries. I apply multi-query only when first-pass retrieval confidence is low (**adaptive**), not on every request.

Measured impact on an internal-docs bot: query condensation alone fixed about 30% of the failures on follow-up questions."

**Follow-ups:**
- *Risk of rewriting?* → Drift: the rewrite changes the meaning. Log both versions and evaluate the rewrites on a sample. Keep the original query in the fusion as one of the inputs.

**Red flags:**
- Sending raw chat messages straight to vector search.
- Using multi-query on every request with no latency awareness.

---

<a id="q18"></a>
## Q18. How do you handle conversational follow-ups ("what about for contractors?") in RAG?
*[Added]*

**What the interviewer is testing:** multi-turn RAG design, specifically combining conversation state with retrieval.

**Strong answer:**

"The follow-up alone retrieves nothing useful: 'what about for contractors?' has no subject. I'd handle it in three pieces:

1. **Query condensation:** a small LLM call takes the last N turns plus the new message and outputs a standalone query: 'What is the leave policy for contractors?'. Condition it on the prior *topic*, not the whole transcript, to keep it cheap.
2. **Carry state explicitly:** keep structured conversation state such as `{topic: leave_policy, entity: contractors, filters: {region: IN}}`. The next turn inherits filters unless overridden. This is more reliable than asking the LLM to re-infer everything.
3. **Decide whether to retrieve again:** if the follow-up is about content already in context ('summarize that in 3 bullets'), reuse the prior context and skip retrieval. If it changes the entity or topic, retrieve again.

Generation then receives: short conversation summary + new retrieved context + the current question.

Edge cases:
- **Topic switch detection:** if the new message is standalone, don't force the old context. The condenser prompt should output the query unchanged in that case.
- **Pronoun resolution errors:** log the condensed query in the trace. This is the first thing I check when a follow-up answer is wrong.

Latency: condensation on a small model is about 150 ms. It can run in parallel with a cheap 'is this standalone?' check."

**Follow-ups:**
- *Why not embed the whole conversation as the query?* → It dilutes the embedding with old topics and hurts precision.
- *How do you evaluate multi-turn?* → Build conversation-level test cases (scripted 3–5 turn dialogs) and measure the per-turn answer accuracy.

**Red flags:**
- No handling of follow-ups at all.
- Appending the entire chat history to the retrieval query.

---

## Rapid-fire recap

- RAG is two pipelines: offline indexing and online retrieval + generation. Answer quality is capped by retrieval quality.
- Chunk by the document's natural structure; 300–800 tokens is a starting point, tuned on a golden set.
- Overlap of 10–20% for fixed-size chunks; little or none for structure-aware ones. Dedupe adjacent chunks.
- Top-K differs per stage: retrieve ~50, rerank to 5–8, then fit the token budget.
- BM25 handles exact terms, dense handles meaning, hybrid is the safe default. Keyword wins on IDs and out-of-domain jargon.
- BM25 = IDF × saturated TF with length normalization (k1 ≈ 1.2–2, b ≈ 0.75).
- RRF: `Σ 1/(60 + rank)`. It fuses ranks, not incomparable scores.
- Bi-encoder for recall at scale, cross-encoder reranker for precision on the top tens.
- Metadata and ACL filters are hard constraints and run during retrieval, never after.
- Citations need provenance stored at ingestion, plus quote verification at answer time.
- Tables: stitch across pages, repeat headers in row chunks, send aggregations to SQL.
- Route scans to OCR or layout models and figures to vision LLMs, watching cost per page.
