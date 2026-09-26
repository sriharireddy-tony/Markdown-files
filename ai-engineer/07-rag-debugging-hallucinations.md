# 07 — RAG Debugging & Hallucinations

What interviewers probe here: whether you debug systematically, separating ingestion, retrieval, context assembly, generation and evaluation, instead of blaming "the model". Senior candidates show they know most "hallucinations" in production are plumbing bugs.

[← Back to index](README.md)

1. [Accuracy dropped 85% → 60% after adding documents](#q1-rag-accuracy-dropped-from-85-to-60-after-adding-documents-how-do-you-find-the-root-cause-systematically)
2. [Correct document retrieved, wrong answer](#q2-the-system-retrieved-the-correct-document-but-still-gave-the-wrong-answer-where-do-you-debug-first)
3. [The chatbot started hallucinating](#q3-the-chatbot-started-hallucinating-what-do-you-check-and-in-what-order)
4. [Reducing hallucination and making the model refuse](#q4-how-do-you-reduce-hallucination-and-make-sure-users-never-get-made-up-answers-including-making-the-model-refuse)
5. [Voice agent invented eight facts](#q5-real-scenario-a-voice-agent-invented-eight-facts-in-a-row-even-though-retrieval-returned-the-right-chunks-at-085-09-similarity-how-would-you-debug-it)
6. [Retrieval failure vs generation failure](#q6-the-final-answer-is-wrong-how-do-you-tell-a-retrieval-failure-from-a-generation-failure)
7. [Faithfulness up, relevancy down](#q7-faithfulness-improves-but-answer-relevancy-drops-why)
8. [Faithfulness 0.95 but users report wrong answers](#q8-faithfulness-is-095-but-users-still-report-wrong-answers-what-could-be-wrong)
9. [Finding the slow stage](#q9-your-rag-isnt-slow-because-of-the-llm-your-pipeline-is-broken-how-do-you-find-the-slow-stage)
10. [Context-window overflow](#q10-how-do-you-handle-context-window-overflow)
11. [Correct yesterday, wrong today](#q11-answers-were-correct-yesterday-and-wrong-today-with-no-code-change-what-happened)
12. ["Correct but too generic"](#q12-users-complain-answers-are-correct-but-too-generic-what-do-you-change)

---

## Q1. RAG accuracy dropped from 85% to 60% after adding documents. How do you find the root cause systematically?
*Source: interview posts*

**What the interviewer is testing:** A structured, stage-by-stage diagnosis driven by data rather than guesses.

**Strong answer:**

"First I'd confirm the drop is real: same eval set, same judge, same model version? If the eval set is unchanged and only the corpus changed, the problem is almost certainly **retrieval competition**. The new documents are outranking the old correct ones.

My steps, in order:

1. **Split the failures.** For each newly failing question, was the gold chunk in the top-K before, and is it now?
   - Gold chunk missing from the top-K → retrieval problem (most likely).
   - Gold chunk present but the answer is wrong → context/generation problem.
2. **See what displaced it.** Which docs now occupy the top-K? Common patterns:
   - **Near-duplicates or older versions** of the same content.
   - **Generic, keyword-heavy docs** (FAQs, glossaries, templates) that are semantically close to everything.
   - **A different domain** in the same index (e.g. new HR docs polluting IT queries).
3. **Check the ingestion of the new batch.** Parse quality (garbled OCR, headers and footers repeated in every chunk, tables flattened), a different chunk size, or **a different embedding model or version** for the new docs.
4. **Check the metrics per slice**: Recall@K on old-domain questions vs new-domain ones.

Fixes, depending on the cause:
- Dedupe and mark superseded versions.
- Metadata filters or routing by domain/collection.
- Hybrid search plus a reranker to reward exact matches.
- Increase candidate K before reranking (e.g. retrieve 50, rerank to 5).
- Re-ingest the new batch with the correct parser and model.

Example: we saw exactly this drop after importing a support-ticket dump. Ticket text was conversational and semantically close to every question, so it dominated the top-5. Separating tickets into their own collection, only queried when the policy retrieval confidence was low, restored accuracy to 83%."

**Follow-ups:**
- *No gold labels?* → Sample 50 failures, have an LLM judge or a human label chunk relevance, then compute Recall@K.
- *How do you prevent this next time?* → Run the golden-set eval as a gate on every corpus ingestion, not just on code changes.

**Red flags:**
- "Switch to a bigger LLM."
- Changing chunk size before diagnosing.
- Not checking whether the eval itself changed.

---

## Q2. The system retrieved the correct document but still gave the wrong answer. Where do you debug first?
*Source: interview posts*

**What the interviewer is testing:** Whether you know that "retrieved" doesn't mean "reached the model in usable form".

**Strong answer:**

"'Retrieved the correct document' is often measured in the retriever logs, not in the **final prompt**. So step one is: **open the exact prompt that was sent to the LLM** for that request, from traces.

Checklist in priority order:
1. **Did the chunk actually reach the prompt?** It may have been dropped by a token-budget trimmer, a reranker cutoff, a deduper, a race condition (retrieval returned after the prompt was built), or an exception swallowed in the context builder.
2. **Is it the right chunk of the right doc?** The doc was right, but the chunk may be the wrong section, or split so that the key sentence or table row lives in the neighboring chunk.
3. **Is it readable?** Tables flattened into word soup, lost headers ('Plan B' values attached to the 'Plan A' row), OCR errors.
4. **Is it buried?** Twelve chunks with the answer in position 7 means lost in the middle. Put the best chunks first or last, and send fewer.
5. **Is it contradicted?** Another chunk (an older version) says something different.
6. **Does the prompt instruct grounding?** The system prompt should say: answer only from the context, cite sources, and say 'I don't know' otherwise. And check that this instruction is actually **attached on the live path**.
7. **Is it a reasoning failure?** Multi-step arithmetic or comparing values across chunks. Fix it by using a tool for the calculation, adding structured extraction, or moving to a stronger model for that route.

I'd reproduce it offline with the exact prompt, then vary one thing at a time: same context plus a stronger model, reordered context, only the gold chunk. That isolates the cause in minutes."

**Follow-ups:**
- *The model ignores the context and uses prior knowledge?* → Stronger grounding instructions, context placed right before the question, and a faithfulness check that forces a regeneration or a refusal.
- *How do you catch this in production?* → Log a hash of each chunk included in the final prompt, and compute the answer's faithfulness against it on a sample.

**Red flags:**
- Assuming the retriever log equals the model input.
- Jumping to fine-tuning.

---

## Q3. The chatbot started hallucinating. What do you check, and in what order?
*Source: interview posts*

**What the interviewer is testing:** Change-driven incident debugging: what changed, and where in the pipeline.

**Strong answer:**

"'Started' means something changed, so I start with **what changed and when**, then walk the pipeline.

1. **Scope it**: all queries or one topic, tenant, language or channel? When did it start? Correlate with deploys, prompt changes, model or provider version updates, and index or ingestion runs.
2. **Pull 20–50 bad traces** and classify each one:
   - Retrieval returned nothing or irrelevant chunks → the model filled the gap from its priors.
   - Retrieval was good, but the context never reached the prompt → plumbing.
   - Context reached the model, but the answer contradicts it → generation/prompt issue.
3. **Check the retrieval layer**: index health (counts, recent failed re-index), embedding model mismatch between query and documents, filters that are too strict (a new ACL filter returns zero chunks), and a vector DB timeout that falls back to empty context.
4. **Check the prompt layer**: was the grounding instruction removed, truncated by a longer new system prompt, or not attached for a new code path? Did the temperature change?
5. **Check the model layer**: a provider silently updated the model alias (`-latest`), or a fallback route switched to a weaker model during an outage.
6. **Check inputs**: a new user population, a new language, or prompt injection in newly ingested docs.

The most common root cause I've seen is **empty or failed retrieval being treated as success**: the model gets no context and answers anyway. The fix is to make an empty context an explicit branch that refuses, asks a clarifying question, or escalates, rather than calling the LLM with nothing."

**Follow-ups:**
- *How do you detect it early next time?* → Online faithfulness sampling, alerts on the 'retrieval returned 0 chunks' rate, and on the refusal rate going to zero.
- *Temporary mitigation?* → Roll back the last change, lower temperature, raise the retrieval threshold, and turn on stricter refusal.

**Red flags:**
- "LLMs hallucinate, it's normal."
- No timeline or change correlation.

---

## Q4. How do you reduce hallucination and make sure users never get made-up answers, including making the model refuse?
*Source: interview posts*

**What the interviewer is testing:** Layered defenses across retrieval, prompting, verification and UX, and an honest view that zero isn't achievable.

**Strong answer:**

"I'd be honest: 'never' isn't achievable with a probabilistic model, but we can make hallucinations rare, detectable and low-impact. Defenses in layers:

**1. Better grounding input**
- Good retrieval: hybrid search, a reranker, fewer and more relevant chunks.
- Live tools for volatile facts (prices, availability) instead of text chunks.

**2. An explicit 'no answer' path**
- If the top reranker score is below the threshold, or zero chunks came back, **don't call the generator with empty context**. Return a scripted response: 'I don't have that detail, let me connect you to the team.'

**3. A grounding prompt**
- 'Answer only using the context. If the answer isn't there, say you don't know. Cite the source ID for every claim.'
- Give the model an allowed way out. Models hallucinate less when refusal is explicitly acceptable.
- Low temperature for factual Q&A.

**4. Verification after generation**
- Citation check: every cited ID exists in the context, and quoted numbers appear verbatim in the cited chunk.
- NLI or an LLM judge for faithfulness on high-risk answers. If the answer is unsupported, regenerate once, then refuse.
- Structured outputs for facts (JSON fields filled from the context), which are easier to validate.

**5. Product and UX**
- Show citations, let users click through, show confidence, and escalate to a human for high-stakes domains.

**6. Monitoring**
- Sampled online faithfulness, user-reported errors and refusal rate. A refusal rate that suddenly hits 0% is as suspicious as one that suddenly hits 40%.

Trade-off: stricter refusal lowers hallucination but raises 'I don't know' rates and user frustration. I tune the threshold on an eval set against both metrics, and per domain. Medical or finance can be strict; casual FAQs can be looser."

**Follow-ups:**
- *Does fine-tuning reduce hallucination?* → It can teach the refusal behavior and format, but it doesn't add reliable facts. Facts come from retrieval.
- *How do you measure hallucination rate?* → Claims not supported by the context divided by total claims, via an LLM judge calibrated on human labels.

**Red flags:**
- "Just add 'do not hallucinate' to the prompt."
- Promising 0%.
- No refusal path.

---

## Q5. Real scenario: a voice agent invented eight facts in a row even though retrieval returned the right chunks at 0.85–0.9 similarity. How would you debug it?
*Source: interview posts*

**What the interviewer is testing:** Whether you check the plumbing before blaming the model, and whether you can tell a clear real-world debugging story.

**Strong answer:**

"This is a real pattern I've seen. A production voice agent for a real-estate client confidently invented a price, a unit configuration, construction status and a bank tie-up, eight facts in one call. None of it was in the knowledge base.

The easy conclusion was 'the model is hallucinating, switch models'. But the retrieval logs showed the **right chunks at 0.85–0.9 similarity**. So retrieval wasn't the problem, and I asked: **did those chunks and our rules actually reach the model on that call?**

I pulled the exact final prompt from the trace for that turn, and found two plumbing bugs:

1. **The anti-hallucination rule was dead code.** The instruction 'Only answer from the provided context; if a detail is missing, say you don't have it' was defined in a prompt module but never attached to the prompt on the live call path. It existed in the codebase and in the tests' version of the prompt, but not in production.
2. **Retrieval ran fire-and-forget.** It was launched asynchronously to save latency, but the prompt was built and sent **before the results came back**. The model answered with **zero context** and filled the gaps from its priors, fluently and confidently. The retriever logged a success a few hundred milliseconds later, which is why the logs looked fine.

Fixes:
- **Await retrieval** (with a timeout) before building the prompt. If it times out, take an explicit fallback branch: 'Let me check that for you' or a handoff, never generating without context.
- **Attach the grounding rules in the one code path** that builds every live prompt, and add a **test that asserts on the final assembled prompt** (the rule is present, and the context block is non-empty when the query needs facts).
- **Log the final prompt's context size** per turn, and alert when factual intents run with 0 context tokens.
- For latency, instead of fire-and-forget, **start retrieval early** (for example, on partial speech transcripts) and play a short filler while waiting, keeping the reply within the ~1–1.2 s budget.

After the fix, the agent says: 'I don't have that detail, let me connect you to our sales team.' That's less impressive in a demo and far better on a real call.

**The lesson:** most of the work in a production RAG system isn't retrieval quality. It's making sure what you retrieved **actually reaches the model**, and that the model **refuses when it didn't**."

**Follow-ups:**
- *Why didn't your tests catch it?* → They tested components (the retriever returns chunks; the prompt template contains the rule), not the assembled live request. Add end-to-end assertions on the real prompt.
- *How do you keep latency low without fire-and-forget?* → Parallelize retrieval with other pre-work, speculatively start on partial transcripts, cache frequent intents, and use a fast model.
- *Why would a model invent eight facts so confidently?* → With no context and a persona prompt saying 'you are the sales assistant for X', the most likely continuation is a plausible sales answer. Priors fill the vacuum.

**Red flags:**
- "Retrieval similarity was high, so it must be the model." Similarity in logs doesn't prove the context was in the prompt.
- Switching models or fine-tuning before reading the actual prompt.
- Fixing it with a stronger 'don't hallucinate' sentence while context is still missing.

---

## Q6. The final answer is wrong. How do you tell a retrieval failure from a generation failure?
*Source: interview posts*

**What the interviewer is testing:** Component-level evaluation and controlled experiments.

**Strong answer:**

"I split the question into two measurable questions:

1. **Was the needed information in the context sent to the model?** (retrieval + context assembly)
2. **Given that context, was the answer correct and faithful?** (generation)

| Context has the answer? | Answer faithful to context? | Diagnosis |
|---|---|---|
| No | — | Retrieval failure (or a data gap: the answer isn't in the corpus at all) |
| Yes | No | Generation failure: ignored context, bad reasoning, lost in the middle |
| Yes | Yes, but still wrong | The source document is wrong or outdated → data problem |
| No, and the model answered anyway | — | Missing refusal behavior |

Techniques:
- **Oracle test**: feed the model only the gold chunk. If it now answers correctly, retrieval or context assembly is the problem. If it still fails, it's the generator or the prompt.
- **Metrics**: context recall/precision for retrieval, faithfulness for generation (RAGAS-style), each computed per question so you can bucket failures.
- **Data-gap check**: search the corpus manually or with BM25 for the answer. Often it simply isn't there, which is a content fix, not an engineering fix.

In my experience the split on a mature system is roughly 50–60% retrieval or context issues, 20–30% generation, and 10–20% data gaps or bad labels. Knowing the split tells you where to invest."

**Follow-ups:**
- *What if the eval label itself is wrong?* → Always review a sample of 'failures'. Often 10%+ are gold-label errors.
- *Tooling?* → Tracing (LangSmith, Langfuse, Phoenix) that shows retrieved chunks, the final prompt and the output side by side.

**Red flags:**
- Only an end-to-end accuracy number, with no component metrics.
- Never considering that the answer isn't in the corpus.

---

## Q7. Faithfulness improves but answer relevancy drops. Why?
*Source: interview posts*

**What the interviewer is testing:** Understanding metric trade-offs and what each metric actually measures.

**Strong answer:**

"Faithfulness asks: **are the answer's claims supported by the context?** Answer relevancy asks: **does the answer address the question?** They can move in opposite directions.

Likely causes:
- **Over-conservative prompting.** After adding 'only use the context', the model refuses more ('I don't have that information') or answers narrowly. Refusals are trivially faithful but score low on relevancy.
- **Extractive copying.** The model pastes the retrieved text verbatim. That's highly faithful, but it answers a neighboring question or buries the answer in irrelevant detail.
- **Retrieval drifted to 'safe' but off-target chunks.** The model faithfully summarizes a related chunk that doesn't answer the question.
- **Hedging and verbosity.** Long, caveated answers dilute relevancy scores.
- **Metric artifact.** RAGAS answer relevancy generates questions from the answer and compares them with the original question. Refusals and generic answers generate very different questions, so the score drops.

How I'd check: segment the eval by refusal vs non-refusal answers. If the drop is concentrated in refusals, look at whether those questions are actually answerable from the corpus (check context recall). If they are, the grounding is too strict or retrieval is missing the answer.

The fix is usually to keep the grounding rule but add 'answer the question directly in the first sentence, then support it', improve retrieval recall, and tune the refusal threshold. The target is both metrics high: faithful and on point."

**Follow-ups:**
- *Which matters more?* → It depends on domain risk. In compliance, prefer faithfulness; in general support, a refusal is also a failure, so track 'answerable question refused' separately.

**Red flags:**
- Treating either metric as the single truth.
- Not knowing how answer relevancy is computed.

---

## Q8. Faithfulness is 0.95 but users still report wrong answers. What could be wrong?
*Source: interview posts*

**What the interviewer is testing:** Understanding that faithfulness only measures consistency with the context, not correctness, and awareness of eval blind spots.

**Strong answer:**

"Faithfulness only says **the answer matches the retrieved context**. It doesn't say the context was right, complete or relevant, or that the eval represents real users. Candidates:

1. **Wrong or stale context.** The model faithfully repeats an outdated policy or a superseded version. Faithful and wrong.
2. **Incomplete context.** The answer is supported by the chunk it saw, but a crucial exception ('except in California') lives in a chunk that wasn't retrieved. Check context recall, not just faithfulness.
3. **Eval set doesn't match production.** The golden set is clean, English and single-hop; real users ask ambiguous, multi-part or multilingual questions. Compare the eval query distribution with a sample of production queries.
4. **Judge weakness.** The LLM judge is lenient, or has the same blind spots as the generator (self-preference). Calibrate it against human labels on 100–200 samples.
5. **Claim granularity.** The metric checks claims it extracts; an unsupported number or a wrong entity in a long answer can slip through.
6. **'Wrong' by user expectation.** The answer is technically correct but not what the user needed (the wrong plan tier, missing the next step). That's a relevancy or UX problem.
7. **Sampling.** 0.95 on 50 examples has a wide confidence interval, and failures may cluster in a high-traffic segment.

Next step: take the user-reported cases, run them through the tracing tool, classify each (stale data, missing chunk, wrong question interpretation, judge miss), and add them to the eval set so they become regression tests."

**Follow-ups:**
- *Which metrics would you add?* → Answer correctness against a reference, context recall, per-segment breakdowns, and human-reviewed samples of production traffic.
- *How do you keep the eval set representative?* → Refresh it monthly with sampled production queries, stratified by intent.

**Red flags:**
- "The metric says 0.95, so users are wrong."
- Equating faithfulness with correctness.

---

## Q9. "Your RAG isn't slow because of the LLM; your pipeline is broken." How do you find the slow stage?
*Source: interview posts*

**What the interviewer is testing:** Instrumentation-first performance debugging.

**Strong answer:**

"I don't guess. I add **distributed tracing with a span per stage**, using OpenTelemetry or an LLM tracing tool:

```
request ─┬─ auth/guardrails        15 ms
         ├─ query rewrite (LLM)   620 ms   <- suspicious
         ├─ embed query            45 ms
         ├─ vector search          30 ms
         ├─ BM25 search            25 ms   (sequential after vector!)  <- fix: parallel
         ├─ rerank 100 docs       480 ms   <- rerank 30 instead
         ├─ build prompt (9k tok)  5 ms    <- long prefill downstream
         └─ LLM generate         2100 ms   (TTFT 900 ms)
```

What I look for:
- **Sequential calls that could run in parallel** (vector + BM25 + metadata DB).
- **Extra LLM calls**: rewriting, grading, multi-query. Each costs 300–800 ms.
- **Oversized context**, which inflates TTFT through prefill. 9k tokens of context versus 3k can double TTFT.
- **Reranker candidate count** and whether it runs on CPU.
- **Cold starts, connection setup** (no pooling, new TLS handshakes per request) and retries on timeouts.
- **P95/P99, not averages.** The tail is often one stage timing out and retrying.

Then I fix the top contributor, re-measure and repeat. In the example above, parallelizing search, skipping the rewrite for self-contained queries, reranking 30 docs and trimming the context took P95 from 4.2 s to 1.8 s, without touching the model."

**Follow-ups:**
- *Which metric matters most to users?* → TTFT for chat, total latency for API or voice, P95 over mean.
- *How do you keep it from regressing?* → Latency budgets per stage, with alerts and a load test in CI.

**Red flags:**
- Optimizing without measurements.
- Reporting only average latency.

---

## Q10. How do you handle context-window overflow?
*Source: interview posts*

**What the interviewer is testing:** Token budgeting and prioritization, not "use a bigger context model".

**Strong answer:**

"I treat the context as a **budget** with explicit allocations, for example on a 32k window:

| Slot | Budget |
|---|---|
| System prompt + rules | 1.5k (cached) |
| Conversation history | 3k (summarized beyond the last N turns) |
| Retrieved context | 6–8k |
| Tool results | 2k (truncated/summarized) |
| Reserved for output | 2k |

Strategies:
- **Count tokens before sending**, with the model's tokenizer, and never let the API silently truncate. Silent truncation often cuts the system prompt or the question.
- **Retrieve less, better**: rerank and keep the top 3–5 chunks; compress by extracting only the relevant sentences.
- **Summarize old conversation turns** and keep recent turns verbatim.
- **Truncate tool outputs**: large JSON gets filtered to the needed fields; logs get summarized.
- **Map-reduce** for genuinely large inputs (summarize a 300-page doc): process chunks, then combine.
- **Priority order when trimming**: drop low-ranked chunks first, then older history, never the system rules or the current question.

Trade-off: longer context models exist, but cost and latency scale with input tokens, and quality degrades on long contexts (lost in the middle). Bigger windows are a fallback, not a strategy."

**Follow-ups:**
- *What happens if you exceed the limit?* → Either an API error or silent truncation, depending on the provider and framework. Both are bugs you should catch in code.
- *Where to place key info?* → The most relevant chunks at the start or end, and the question after the context.

**Red flags:**
- "Just use a 1M-token model."
- No token counting.

---

## Q11. Answers were correct yesterday and wrong today, with no code change. What happened?
*[Added]*

**What the interviewer is testing:** Awareness of everything that changes outside your codebase.

**Strong answer:**

"'No code change' doesn't mean no change. The system has several moving parts outside the repo:

1. **Model**: a provider updated the model behind an alias (`gpt-x-latest`, `-preview`), or traffic was routed to a fallback model during an incident. Fix: pin dated model versions and log the model ID returned in each response.
2. **Data**: an ingestion job ran overnight, with new docs outranking old ones, a failed re-index leaving a partial index, or a parser version bump.
3. **Config/feature flags**: prompt templates stored in a DB or config service, changed thresholds, a new top-K.
4. **Dependencies**: a floating library version (e.g. a framework minor release changed default chunking or prompt templates), or a container rebuilt with new packages.
5. **External tools/APIs**: a tool now returns a different schema or empty results.
6. **Traffic**: new users, a new language, a marketing campaign bringing new question types.
7. **Caches**: a bad answer was cached and is being served to everyone.

I'd diff traces from yesterday and today for the same question: model ID, retrieved chunk IDs, prompt hash, and tool outputs. Whichever differs is the culprit. Prevention: pinned versions everywhere, versioned prompts and configs, index versions logged per request, and a daily canary eval run against production."

**Follow-ups:**
- *What's a daily canary eval?* → 50–100 fixed questions run against production daily, with an alert when the score drops by more than X.

**Red flags:**
- "Nothing changed, so it must be random."
- No versioning of models, prompts or indexes.

---

## Q12. Users complain answers are "correct but too generic". What do you change?
*[Added]*

**What the interviewer is testing:** Moving from correctness to usefulness: personalization, specificity and context.

**Strong answer:**

"Generic answers usually mean the model lacks **specific context** or has been told to be too cautious.

Check and fix:
- **Retrieval granularity**: chunks retrieved from overview or FAQ pages instead of the specific procedure. Boost specific doc types, and use metadata (product, plan, region) to filter.
- **User context missing**: the model doesn't know the user's plan, role, region or account state. Inject relevant profile attributes and live account data through tools.
- **Prompt style**: over-hedged system prompts ('be general, avoid specifics'). Ask for concrete steps, the exact values from the context, and a next action.
- **Query understanding**: ambiguous questions get generic answers. Ask one clarifying question when intent confidence is low.
- **Few-shot examples** of the target specificity.
- **Measure it**: add a 'specificity/actionability' rubric to the LLM judge, calibrated on examples of good and bad answers, and track it alongside correctness.

Example: a benefits assistant answered 'contact HR for leave details' until we injected the employee's location and employment type from the HRIS. The answer became 'You have 18 days; 6 carry over until March 31', and CSAT rose noticeably."

**Follow-ups:**
- *Privacy concerns with injecting user data?* → Inject the minimum needed fields, respect permissions, and never log them unredacted.

**Red flags:**
- "Use a bigger model."
- Not considering user context or retrieval granularity.

---

## Rapid-fire recap

- An accuracy drop after adding documents is almost always retrieval competition: duplicates, generic docs, mixed domains, or a mismatched embedding model.
- "Retrieved" is not "in the prompt". Always read the exact final prompt from traces.
- An empty or failed retrieval must be an explicit branch that refuses or escalates, never a silent generation.
- Fire-and-forget retrieval produces fast, confident hallucinations.
- Test the assembled live prompt, not just the components.
- Oracle test: gold chunk only. If the answer is right, fix retrieval; if still wrong, fix generation.
- Faithful isn't correct: stale or incomplete context produces faithful wrong answers.
- Refusals inflate faithfulness and deflate relevancy. Segment metrics by refusal.
- Hallucinations can be made rare and detectable, not impossible. Layer retrieval, grounding, verification and UX.
- Trace a span per stage; the tail latency usually hides in extra LLM calls and sequential I/O.
- Budget the context window explicitly; never let the provider truncate silently.
- "No code change" still leaves model aliases, data, configs, dependencies and caches that can change.
