# 08 — Evaluation

**What interviewers probe here:** can you *prove* a change made the system better, instead of eyeballing a few outputs? Expect questions on retrieval and generation metrics, LLM-as-a-judge and its biases, statistical confidence, and running evaluation continuously in production. This is the area candidates most often skip, so doing it well stands out.

[← Back to index](README.md)

## Questions

1. [How would you measure whether your RAG system actually improved?](#q1)
2. [Retrieval metrics: Precision@K, Recall@K, MRR, Hit Rate, nDCG](#q2)
3. [RAGAS: faithfulness, answer relevancy, context precision, context recall](#q3)
4. [How do you measure the accuracy of the LLM's generation?](#q4)
5. [LLM-as-a-Judge: design, representative examples, calibration](#q5)
6. [Detecting verbosity, position and self-preference bias in judges](#q6)
7. [Detecting eval gaming (overfitting to the judge)](#q7)
8. [An eval system that catches silent regressions across dozens of prompts](#q8)
9. [Continuous evaluation against a shifting production distribution](#q9)
10. [How many samples before you trust an eval number?](#q10)
11. [Evaluating a fine-tuned LLM against its base model](#q11)
12. [Evaluating multimodal RAG beyond RAGAS](#q12)
13. [Monitoring hallucination rate automatically](#q13)
14. [Improving the system from user feedback](#q14)
15. [Building a ground-truth dataset; where human eval fits](#q15)
16. [Evaluating agents: tool selection, planner, multi-agent levels](#q16)
17. [Offline metrics improved but online CSAT didn't move](#q17)

---

<a id="q1"></a>
## Q1. How would you measure whether your RAG system actually improved?
*Source: interview posts*

**What the interviewer is testing:** whether you have an evaluation methodology or just a feeling that "answers look better".

**Strong answer:**

"I'd first pin down what 'improved' means, because a RAG system can get better at retrieval and worse at answering. I measure three layers separately, then check them against the business outcome.

1. **Retrieval:** on a labeled set of questions with known relevant chunks, I track Recall@K (did the right evidence come back at all?) and MRR or nDCG (was it near the top?).
2. **Generation:** faithfulness (is every claim supported by the context?), answer correctness against a reference answer, and answer relevancy.
3. **System:** P95 latency, cost per query and refusal rate. A change that raises quality by 2 points but doubles latency may not count as an improvement.

The process is:
- A **frozen golden set**, for example 300–500 questions stratified by intent: FAQ, multi-hop, table lookups, questions the system should refuse on, and recent-doc questions.
- **Run baseline and candidate on the same set**, with the same judge and settings, and compare them *paired*, question by question.
- **Check significance.** A 1-point gain on 200 questions is noise (see Q10).
- **Read the diff.** I look at the questions that flipped from correct to wrong, not just the average.
- **Online confirmation:** a shadow run or A/B test on real traffic, tracking thumbs-up rate, escalation or deflection rate, and follow-up rephrasing rate.

Example: when we added a reranker, Recall@20 didn't change (it can't, since the reranker only reorders), but MRR went from 0.52 to 0.71 and answer correctness went from 74% to 81% on 400 paired questions (p < 0.01). P95 latency grew by 180 ms. We accepted that trade."

**Follow-ups:**
- *Why measure retrieval separately?* → If the end-to-end score drops, you need to know which stage broke. You can't fix what you can't localize.
- *What if you have no labels?* → Bootstrap a set: sample real queries, draft references with an LLM, have domain experts review them, and use reference-free metrics (faithfulness) in the meantime.
- *How often do you refresh the golden set?* → Keep a frozen core for comparability, and add a rolling slice sampled from recent production traffic each month.

**Red flags:**
- "We tested it on a few questions and it looked better."
- Reporting one averaged number, with no confidence interval and no per-segment view.
- Changing the eval set or the judge prompt between baseline and candidate.

---

<a id="q2"></a>
## Q2. Explain retrieval metrics: Precision@K, Recall@K, MRR, Hit Rate, nDCG. How do you measure retriever accuracy?
*Source: interview posts*

**What the interviewer is testing:** that you know what each metric rewards, and which one fits a RAG system.

**Strong answer:**

"For each query I need the set of relevant chunk IDs, which is the ground truth. Then:

| Metric | Formula (per query, then averaged) | What it tells you |
|---|---|---|
| **Precision@K** | relevant in top K / K | How much noise goes to the LLM |
| **Recall@K** | relevant in top K / total relevant | Did we fetch the needed evidence at all? |
| **Hit Rate@K** | 1 if ≥1 relevant in top K, else 0 | Recall for single-answer questions |
| **MRR** | mean of 1 / rank of first relevant | How high the first good chunk sits |
| **nDCG@K** | DCG/IDCG, where DCG = Σ rel_i / log2(i+1) | Ranking quality with graded relevance |

For RAG, **Recall@K is the ceiling metric**. If the evidence isn't retrieved, the LLM can't answer faithfully. **MRR and nDCG matter because of limited context and 'lost in the middle'**: the best chunk should be at the top. Precision@K matters for cost and distraction. Low precision means the LLM is fed junk that can pull the answer off course.

To measure it: build (query, relevant chunk IDs) pairs from expert labeling, from user click logs, or synthetically, by generating questions *from* a chunk so that chunk is the known positive. Evaluate at several K values, such as 5, 10 and 20. One gotcha: if you re-chunk, chunk IDs change, so I label at the **document or passage-span level** and count a retrieved chunk as relevant if it overlaps the gold span.

Example: our Hit Rate@5 was 0.91 but Recall@5 on multi-hop questions was only 0.48. The system found one of the two needed passages but not the other. That told me the issue was multi-hop retrieval, not the embedder."

**Follow-ups:**
- *Precision or recall first?* → Recall first, then use a reranker to improve precision within the top K.
- *Why nDCG over MRR?* → When relevance is graded (perfect, partial, irrelevant) or there are several relevant docs.
- *How do you pick K for the metric?* → Match the K you actually send to the LLM after reranking, plus the larger candidate K before reranking.

**Red flags:**
- Confusing Precision@K with the accuracy of the final answer.
- Measuring retrieval only indirectly through answer quality.
- Not knowing that a reranker can't improve Recall@K for the candidate set it's given.

---

<a id="q3"></a>
## Q3. Explain RAGAS: faithfulness, answer relevancy, context precision, context recall. How is each computed, and how do you use RAGAS on a response?
*Source: interview posts*

**What the interviewer is testing:** whether you understand how the metrics are computed internally, not just their names.

**Strong answer:**

"RAGAS is an open-source framework that uses an LLM (plus embeddings) to score RAG outputs. The four core metrics each isolate a different failure:

| Metric | Inputs | How it's computed | Catches |
|---|---|---|---|
| **Faithfulness** | answer, contexts | LLM splits the answer into atomic claims; each claim is checked against the context; score = supported claims / total claims | Hallucination beyond the context |
| **Answer relevancy** | question, answer | LLM generates N questions the answer would answer; mean cosine similarity to the real question | Off-topic or evasive answers |
| **Context precision** | question, contexts, reference | Each chunk judged relevant or not; rank-weighted precision (mean of Precision@k at relevant positions) | Relevant chunks ranked low |
| **Context recall** | contexts, reference | Reference answer split into claims; fraction attributable to the retrieved context | Missing evidence |

Faithfulness and answer relevancy are **reference-free**, so you can run them on production traffic. Context recall and precision (in its reference-based variant) need a **reference answer**, so they run on the golden set.

```python
# ragas >= 0.2 style API (older versions used datasets.Dataset + lowercase metric objects)
from ragas import EvaluationDataset, evaluate
from ragas.metrics import (Faithfulness, ResponseRelevancy,
                           LLMContextPrecisionWithReference, LLMContextRecall)
from ragas.llms import LangchainLLMWrapper
from ragas.embeddings import LangchainEmbeddingsWrapper
from langchain_openai import ChatOpenAI, OpenAIEmbeddings

samples = [{
    "user_input": "What is the notice period for contractors?",
    "retrieved_contexts": ["Contractors must give 15 days notice ...", "..."],
    "response": "Contractors need to give 15 days notice.",
    "reference": "15 days written notice.",
}]
ds = EvaluationDataset.from_list(samples)

judge = LangchainLLMWrapper(ChatOpenAI(model="gpt-4o", temperature=0))
emb = LangchainEmbeddingsWrapper(OpenAIEmbeddings())

result = evaluate(ds, metrics=[Faithfulness(), ResponseRelevancy(),
                               LLMContextPrecisionWithReference(), LLMContextRecall()],
                  llm=judge, embeddings=emb)
print(result.to_pandas())   # per-sample scores, not just the mean
```

How I use it: I look at per-sample scores and the claim-level verdicts, not only the mean. Low context recall with high faithfulness means retrieval is the problem. High recall with low faithfulness means generation or the prompt is the problem.

Caveats: RAGAS scores depend on the judge LLM and version, so I pin both. I also spot-check about 50 samples against human labels before trusting it."

**Follow-ups:**
- *Can faithfulness be 1.0 while the answer is wrong?* → Yes, if the retrieved context itself is wrong or outdated. Faithfulness measures grounding, not truth.
- *Why can answer relevancy be gamed?* → It's embedding similarity, so an answer that restates the question scores well. Pair it with correctness.
- *Alternatives?* → TruLens (RAG triad), DeepEval (pytest-style), Arize Phoenix, or custom judges.

**Red flags:**
- "RAGAS tells you whether the answer is correct."
- Not knowing which metrics need ground truth.
- Reporting RAGAS scores without saying which judge model produced them.

---

<a id="q4"></a>
## Q4. How do you measure the accuracy of the LLM's generation, and with which metrics?
*Source: interview posts*

**What the interviewer is testing:** that you match the metric to the task instead of reaching for BLEU or ROUGE by reflex.

**Strong answer:**

"'Accuracy' depends on the output type, so I pick metrics per task:

| Output type | Metric |
|---|---|
| Classification / extraction / routing | Exact match, per-field accuracy, precision/recall/F1 |
| Structured output | Schema validity rate, field-level exact match |
| Short factual answers | Exact match / normalized match, or judge-based correctness vs reference |
| Long-form answers | LLM-judge on a rubric (correctness, completeness, faithfulness, tone), pairwise preference win rate |
| Code / SQL | Execution-based: tests pass, result-set match |
| Summaries | Claim-level faithfulness + coverage of key points |

BLEU and ROUGE measure n-gram overlap. They correlate poorly with quality for open-ended generation, and a correct paraphrase gets penalized. I use them only as cheap sanity checks. BERTScore is a bit better but still doesn't capture factual correctness.

For long-form answers I prefer **decomposed rubrics**. Rather than asking a judge for a 1–10 score, I ask binary questions: 'Does the answer state the correct notice period?', 'Does it contain any claim not in the context?', 'Does it cite a source?' Binary checks are more reliable and easier to calibrate against humans.

I also always track **safety and format metrics** alongside quality: refusal correctness (refuses when it should, answers when it can), PII leakage and policy violations.

Example: for a support bot we defined 'correct' as three checks: right policy fact, no unsupported claim, and correct escalation behavior. A response had to pass all three. Headline accuracy went from a flattering 92% (judge 1–10 score of 7 or more) to an honest 78%, which pointed us at the escalation failures."

**Follow-ups:**
- *Pointwise or pairwise judging?* → Pairwise is more sensitive for comparing two versions. Pointwise with a rubric is better for tracking absolute quality over time.
- *How do you handle multiple valid answers?* → Use several references, or have the judge check against key facts rather than exact wording.

**Red flags:**
- "We use BLEU to measure chatbot accuracy."
- A single vague 1–10 'quality' score with no rubric.
- Ignoring whether the model correctly refuses.

---

<a id="q5"></a>
## Q5. LLM-as-a-Judge: how do you design it, pick representative examples and calibrate against humans?
*Source: interview posts*

**What the interviewer is testing:** whether you treat the judge as a model that itself needs validating.

**Strong answer:**

"An LLM judge is a classifier, so I build and validate it like one.

**Design:**
- **Narrow, binary or low-cardinality criteria**, one criterion per judge call. For example, 'Is every claim supported by the context? yes/no', not 'rate quality 1–10'.
- Give the judge **the rubric, the context and the reference** where available. Ask for a short reasoning *before* the verdict, and use structured output.
- Use a **strong model at temperature 0, pinned to a version**, and preferably a different model family from the generator to reduce self-preference.
- For comparisons, use **pairwise** judging with order randomized (both orders, see Q6).

**Selecting representative examples** (the Round-1 question in the posts):
- **Stratified sampling** from production logs: cluster queries by embedding or intent, then sample across clusters so rare-but-important intents aren't drowned out by the head.
- Deliberately include **hard and edge cases**: known past failures, adversarial inputs, questions the system should refuse on, long multi-hop questions.
- Include **both clear passes and clear fails**. A calibration set that's all good answers can't reveal false positives.
- About 100–300 examples is usually enough to calibrate a binary judge.

**Calibration:**
- Two or more humans label the calibration set. Measure **human-human agreement** (Cohen's kappa) first. If humans only agree at κ = 0.5, the rubric is ambiguous, and I fix the rubric before blaming the judge.
- Then measure **judge-human agreement**: kappa plus precision/recall on the 'fail' class, since missing failures is the expensive error.
- Iterate the judge prompt on a *dev* split and report agreement on a *held-out* split, so I don't overfit the judge prompt.
- Re-calibrate whenever the judge model, the rubric or the product domain changes.

Example: our first faithfulness judge had κ = 0.58 with humans. It passed answers that hedged ('it may be 15 days'). Adding an explicit rule plus two few-shot examples brought κ to 0.81 on the held-out set, above our human-human κ of 0.77, so we trusted it for regression gating."

**Follow-ups:**
- *Is a judge from the same model family a problem?* → It can inflate scores through self-preference. Use a different family or validate against humans.
- *Cost control?* → Use a small judge for high-volume checks and escalate uncertain cases to a large judge. Sample production traffic instead of judging 100%.
- *Can you fine-tune a judge?* → Yes, once you have a few thousand human labels. It gives cheaper and more consistent small judges.

**Red flags:**
- Using the judge's scores without ever comparing to human labels.
- A single catch-all quality score.
- Calibrating and reporting on the same examples.

---

<a id="q6"></a>
## Q6. How do you detect verbosity bias (and position bias and self-preference) in an LLM judge?
*Source: interview posts*

**What the interviewer is testing:** knowledge of known judge biases, and the experiments that expose each one.

**Strong answer:**

"Each bias gets a controlled experiment where the only thing that changes is the suspected cause.

**Verbosity bias**, where the judge prefers longer answers:
- **Correlation check:** plot judge score against answer length (tokens) on the calibration set, and compare it to human score against length. If the judge's slope is clearly positive and the humans' isn't, that's bias.
- **Controlled padding test:** take answers humans rated correct, create a version padded with fluent but information-free text or repeated caveats, and judge both. A fair judge scores them equally or penalizes the padding. A biased one prefers the padded version, and I report that preference rate.
- **Length-matched pairs:** in pairwise evals, compare win rate on pairs within ±10% length versus pairs with a large length gap.

**Position bias** in pairwise judging:
- Run every pair **in both orders**. Consistency rate = fraction where the verdict survives the swap. If A wins as the first answer 60% of the time regardless of content, that's position bias. The fix is to always judge both orders and count inconsistent pairs as ties.

**Self-preference:**
- Compare judge-vs-human agreement on outputs from the judge's own model family against other families. A higher false-pass rate on its own outputs means bias. The fix is to use a different-family judge or an ensemble.

**Mitigations:**
- Rubrics that say explicitly 'do not reward length; penalize unsupported or irrelevant content'.
- Decomposed binary criteria, which are much less sensitive to length than holistic scores.
- Length-controlled win rates (a regression that controls for length, as in length-controlled AlpacaEval).
- An ensemble of judges from different families.

Example: on a padding test, our pairwise judge preferred the padded answer 68% of the time. After switching to a claim-level rubric, that fell to 12%, close to the human rate of 9%."

**Follow-ups:**
- *Why does verbosity bias matter for product quality?* → If you optimize prompts against a biased judge, your answers get longer and worse (see Q7).
- *Other biases?* → Sycophancy toward confident tone, style over substance, and deference to authority claims inside the answer.

**Red flags:**
- "Just tell the judge to be unbiased."
- Running pairwise judging in only one order.
- Not knowing that position bias exists.

---

<a id="q7"></a>
## Q7. How do you detect eval gaming, where a model overfits to the judge?
*Source: interview posts*

**What the interviewer is testing:** that you understand Goodhart's law as it applies to automated evaluation.

**Strong answer:**

"Gaming happens when we optimize prompts, fine-tunes or RL against a judge, and the system learns what the judge rewards rather than what users need. Examples include longer answers, confident tone, stuffing in keywords from the rubric, and restating the question to boost relevancy.

**Detection signals:**
1. **Divergence between judge and humans over time.** I keep a small, continuously refreshed human-labeled audit set (for example 100 samples a week). If the judge score climbs while human-rated quality stays flat or drops, the system is gaming the judge.
2. **Divergence between judges.** I optimize against judge A but also report scores from a held-out judge B from a different family, plus rule-based or execution-based metrics. A big gain on A with no gain on B is a strong signal.
3. **Distribution shifts in output features:** average length, number of hedges or disclaimers, rubric keyword frequency, citation count with no accurate citations. Sudden shifts that line up with judge gains are suspicious.
4. **Online metrics:** if offline judge scores improve but thumbs-up rate, resolution rate or escalation don't, the offline signal is being gamed (see Q17).
5. **Adversarial probes:** known-bad answers dressed up in the judge's preferred style. If they start passing, the judge has become exploitable.

**Prevention:**
- **Separate the optimization judge from the reporting judge**, and keep the reporting judge's prompt and examples out of reach of whoever is iterating on prompts.
- Use held-out test sets and rotate part of them regularly.
- Prefer **verifiable metrics** where possible (execution, exact match, citation-span verification).
- Add a length or verbosity penalty when doing RL or best-of-N against a judge.

Example: a team tuned the system prompt to 'maximize helpfulness' against a judge. Judge scores rose 9 points in two weeks, but human audit ratings fell 3 points. Answers had grown 70% longer with generic caveats. Reverting to the rubric and adding a held-out judge caught the next attempt."

**Follow-ups:**
- *Doesn't a human audit set get gamed too?* → Less so if it's refreshed with new production samples and the labelers are blind to which version produced each answer.
- *Can you use the same judge for training and eval?* → Avoid it. Using it for both is exactly how gaming goes unseen.

**Red flags:**
- "If the judge score goes up, the system is better."
- Having no human signal in the loop at all.
- Iterating prompts directly on the test set.

---

<a id="q8"></a>
## Q8. Design an eval system that catches silent regressions across dozens of prompts and features.
*Source: interview posts*

**What the interviewer is testing:** eval as infrastructure, meaning CI for AI behavior, and not as a one-off notebook.

**Strong answer:**

"Silent regressions come from changes whose impact nobody measured: a prompt edit in feature A that breaks a shared template used by feature B, a provider updating a model version, an embedding change, a new retrieval filter. So the design principle is that **every change that can alter behavior goes through the same gate**.

**Components:**

1. **Eval registry.** Each feature owns a suite: a dataset of 50–500 cases, graders and thresholds. Graders are layered from cheap to expensive:
   - deterministic assertions (schema valid, contains a citation, no PII, refuses on out-of-scope input)
   - reference-based checks (exact or field match)
   - LLM-judge rubrics (calibrated, see Q5)
2. **Versioned everything.** Prompts, model IDs, retrieval config and tool schemas live in a config or prompt registry with versions. The eval run records the full config hash, so any score is reproducible.
3. **Dependency mapping.** Map which features use which prompts, templates, models and indexes. A change to a shared component triggers the suites of *all* dependents, not just the feature being edited.
4. **CI gating:**
   - On PR: a fast smoke subset (about 10% of cases, deterministic plus a small judge) in under 5 minutes.
   - Pre-merge or nightly: full suites, compared paired against the main-branch baseline, with a statistical test per suite (see Q10).
   - Block on significant regressions beyond a threshold, for example more than 2 points with p < 0.05 on any Tier-1 metric. Warn on smaller ones.
5. **Per-case diffs**, not just averages. The report lists cases that flipped from pass to fail, because an average can hide 20 new failures offset by 20 new passes.
6. **Scheduled runs with no code change**, nightly against pinned *and* 'latest' model aliases. This catches provider-side model updates and index drift.
7. **Dashboard over time** per feature and metric, with alerting.

**Trade-offs:** full suites with LLM judges cost money. With 40 features × 300 cases × 3 judge calls = 36k calls per run, I run the full set nightly and a subset on each PR, and cache judge verdicts keyed on (input, output, judge version).

Example: a shared 'tone' snippet was edited for the sales bot. The dependency map triggered the HR bot's suite, which showed the refusal-on-legal-questions rate falling from 97% to 81%. The PR was blocked. Without the map, it would have shipped."

**Follow-ups:**
- *Where do cases come from?* → Seed cases plus every production incident converted into a regression case (the 'bug becomes a test' rule).
- *Flaky judges causing false blocks?* → Use temperature 0, majority vote over 3 runs for borderline cases, and statistical thresholds rather than any-drop-fails.

**Red flags:**
- One global eval set for all features.
- Evals run manually 'before big releases'.
- Not versioning prompts and model IDs together with the results.

---

<a id="q9"></a>
## Q9. How do you run continuous evaluation against a live, shifting production distribution?
*Source: interview posts*

**What the interviewer is testing:** online evaluation, drift awareness, and closing the loop back to offline sets.

**Strong answer:**

"A static golden set goes stale because users change what they ask. New products, policies and seasons all shift the query mix. So I run an **online evaluation loop**:

1. **Sample production traffic** continuously, for example 1–5% of requests, stratified by intent, tenant and language, with oversampling of low-confidence or negatively rated interactions. Scrub PII before evaluation.
2. **Reference-free scoring** on the sample: faithfulness to retrieved context, answer relevancy, refusal appropriateness, format and safety checks. No ground truth is needed.
3. **Cheap signals on 100% of traffic:** thumbs up/down, rephrase-within-60s rate, escalation to a human, session abandonment, 'I don't know' rate, citation click-through, latency and cost.
4. **Drift detection on inputs:** embed queries and track the cluster distribution week over week (for example with KL divergence or population stability index over clusters). Alert when new clusters appear, such as a new product launch generating questions the knowledge base can't answer. Also watch the retrieval score distribution: a falling top-1 similarity means queries are drifting away from the corpus.
5. **Human review queue:** a weekly batch of about 100 samples, prioritized by disagreement between signals (judge passed but the user gave a thumbs-down). This is also how the judge stays calibrated.
6. **Feed back to offline:** new clusters and confirmed failures are labeled and added to the golden set as a 'recent' slice. The frozen core stays fixed for comparability over time.

**Trade-offs:** judging 5% of 200k queries a day is 10k judge calls a day. A small judge with escalation keeps that affordable. Online scores are noisier and have no reference, so I use them for **trend and alerting**, and I don't treat them as absolute truth.

Example: after a pricing change, our faithfulness score stayed high but the new 'pricing' cluster grew from 3% to 14% of traffic, with a thumbs-down rate of 31%. The knowledge base had old pricing, so the bot was faithful to the wrong documents. The drift alert on that cluster caught it within 2 days."

**Follow-ups:**
- *How do you avoid comparing apples to oranges week over week?* → Report metrics per cluster, and use mix-adjusted (re-weighted) aggregates.
- *Privacy?* → PII redaction before logging, consent and retention policies, and tenant-level opt-outs.

**Red flags:**
- "We evaluate once before launch."
- Relying only on thumbs up/down, which is sparse and biased toward angry users.
- No path from production failures back into the test set.

---

<a id="q10"></a>
## Q10. Build statistical confidence into an eval result: how many samples before you trust a number?
*Source: interview posts*

**What the interviewer is testing:** basic statistics applied to eval: confidence intervals, power and paired comparisons.

**Strong answer:**

"An eval score is an estimate with uncertainty, so I always report a confidence interval, and I size the set for the smallest difference I care about.

**Confidence interval for a pass rate** (normal approximation to the binomial):

CI = p ± 1.96 · sqrt(p(1−p)/n)

With p = 0.80:

| n | Margin (95%) |
|---|---|
| 100 | ±7.8 pts |
| 200 | ±5.5 pts |
| 500 | ±3.5 pts |
| 1,000 | ±2.5 pts |

So on 200 questions, '80% vs 83%' is indistinguishable. For small n or p near 0 or 1, I use a Wilson interval instead.

**Detecting a 3-point difference between two versions:**

*Unpaired* (different question sets, or an A/B test on traffic), with α = 0.05 and power 80%:

n per arm ≈ (1.96 + 0.84)² · 2 · p(1−p) / d² = 7.84 · 2 · 0.16 / 0.03² ≈ **2,800 per arm**

*Paired* (both versions run on the same questions), which is far more efficient. Only questions where the versions disagree carry information (McNemar's test). If about 10% of questions are discordant and the true difference is 3 points:

n ≈ (1.96·sqrt(0.10) + 0.84·sqrt(0.10 − 0.03²))² / 0.03² ≈ **870 questions**

That's roughly a third of the unpaired requirement, which is why I always evaluate candidate and baseline on the *same* set.

**For non-binary metrics** (judge scores, RAGAS means), I use a **paired bootstrap**: resample question indices with replacement 10,000 times, recompute the mean difference each time, and take the 2.5th–97.5th percentile. If the interval excludes 0, the difference is significant.

**Other hygiene:**
- **Multiple comparisons:** testing 20 metrics at p < 0.05 will produce about one false win. Pre-declare a primary metric, or apply a Holm or Bonferroni correction.
- **Judge noise:** if the judge is stochastic, average multiple runs, or treat the judge as a source of variance.
- **Segment sizes:** a 500-question set with 40 'table' questions gives the table segment a margin of about ±12 pts. Be careful drawing conclusions per segment.

My rule of thumb: about 300–500 paired questions to detect roughly 5-point changes, about 1,000 or more for 3 points, and always show the interval."

**Follow-ups:**
- *We only have 150 labeled examples. Now what?* → Report wide intervals honestly, only act on large effects, and grow the set by converting production failures into cases.
- *Stopping an A/B test early?* → Peeking inflates false positives. Use a fixed horizon or sequential testing (for example SPRT or alpha spending).

**Red flags:**
- "We got 84% vs 82%, so the new version is better," on 100 questions.
- Comparing versions on different question sets.
- Not knowing what a confidence interval is.

---

<a id="q11"></a>
## Q11. How do you evaluate a fine-tuned LLM against its base model?
*Source: interview posts*

**What the interviewer is testing:** that you check both the target gain *and* regressions everywhere else.

**Strong answer:**

"I check three things: did it learn the target task, did it forget anything else, and is it worth the operational cost?

1. **Target-task eval** on a held-out test set that was never in training data. I dedupe by near-duplicate check, not just exact match, because leakage through paraphrased duplicates is common. I use the same metrics as the task demands: field F1 for extraction, rubric judge for style, execution accuracy for SQL.
2. **Fair baselines.** I compare against:
   - the base model zero-shot
   - the base model with a *good* few-shot prompt, and RAG if relevant

   Many fine-tunes beat a lazy zero-shot baseline but not a well-prompted one. That second comparison is what justifies the fine-tune.
3. **Regression and forgetting checks:**
   - general capability benchmarks relevant to the product (instruction following, reasoning, a small MMLU-style slice)
   - safety evals: refusal behavior, jailbreak resistance, toxicity. Fine-tuning often weakens safety alignment even on benign data.
   - format adherence and tool-calling ability if the product uses them
4. **Robustness:** out-of-distribution inputs, such as different phrasing, longer inputs and other languages, to catch overfitting to the training template.
5. **Operational:** latency, throughput, cost per 1k requests and serving complexity.
6. **Online:** shadow deployment, then an A/B test on real traffic.

Everything is compared paired on the same inputs, with confidence intervals (see Q10).

Example: a LoRA fine-tune of an 8B model for invoice extraction reached 94.1% field-F1 against 86.3% for base few-shot and 95.0% for a frontier API. But the general instruction-following slice dropped 6 points, and the model started emitting JSON for non-extraction prompts. We shipped it only behind the extraction route, at about a fifth of the frontier API's cost."

**Follow-ups:**
- *How do you detect training-data contamination?* → Check n-gram or embedding overlap between train and test sets, and check whether performance on 'new' data drops sharply.
- *Loss went down but quality didn't. Why?* → Loss measures token imitation, not task success. Overfitting to the format is also possible. Always evaluate generations.

**Red flags:**
- Using validation loss as the evaluation.
- Comparing only against the zero-shot base model.
- No safety or general-capability regression check.

---

<a id="q12"></a>
## Q12. How would you evaluate multimodal RAG beyond RAGAS?
*Source: interview posts*

**What the interviewer is testing:** whether you can extend the evaluation mindset to images, tables and charts, where text metrics break.

**Strong answer:**

"Multimodal RAG adds failure points that text metrics don't see: parsing or OCR errors, lost table structure, wrong image retrieval, and misreading charts. So I evaluate **per stage and per modality**.

1. **Ingestion and extraction quality**, often the biggest error source:
   - OCR: character and word error rate (CER/WER) on a labeled page sample.
   - Tables: cell-level accuracy, and structure metrics like TEDS (tree-edit-distance similarity), which checks that headers and cell relations are preserved, including tables that span pages.
   - Layout: were reading order and section boundaries preserved?
   - Figures: are captions and chart descriptions correct? A human or VLM check on a sample.
2. **Cross-modal retrieval:** Recall@K and MRR where the gold item may be a text chunk, a table or an image region. Report them **separately per modality**, because text retrieval often looks great while table and image recall lags.
3. **Grounded generation:**
   - Faithfulness *to the visual source*: the judge needs the image or table itself, not just extracted text. I use a VLM judge, validated against humans, or check against structured ground truth.
   - **Numeric correctness** for chart and table questions: exact or tolerance-based match on the extracted value. That's deterministic and doesn't need a judge.
   - **Citation localization:** does the answer point to the right page and bounding box? Measure IoU or page-match rate.
4. **Question-type stratification:** text-only, table lookup, table aggregation, chart reading, image-content questions and cross-modal questions ('does the diagram match the spec text?').
5. **End-to-end:** human expert review on a sample, since VLM judges misread charts too.

Example: text faithfulness was 0.93, but on table-aggregation questions numeric correctness was only 61%. Tracing the failures showed that a table spanning two pages was split into two chunks and the second lost its headers. After switching to table-aware ingestion that repeats headers per chunk, numeric correctness rose to 84%."

**Follow-ups:**
- *Is a VLM-as-judge reliable?* → Only after calibration. VLMs make their own chart-reading mistakes, so prefer structured ground truth for numeric questions.
- *How do you build ground truth cheaply?* → Generate questions from known table cells or chart data series, where the answer is known programmatically.

**Red flags:**
- Running RAGAS on OCR text and calling the multimodal evaluation done.
- Not evaluating extraction separately.
- Using an LLM judge for numeric answers that could be checked deterministically.

---

<a id="q13"></a>
## Q13. How would you monitor hallucination rate automatically?
*Source: interview posts*

**What the interviewer is testing:** that you can turn hallucination from an anecdote into a production metric.

**Strong answer:**

"First I define it operationally. In RAG, a hallucination is **a claim in the answer that isn't supported by the retrieved context or by tool results**. That's measurable without knowing the ground truth.

**Pipeline:**
1. **Log the full trace** for every request: query, retrieved chunks with IDs, the *final prompt actually sent*, tool outputs and the answer. The final prompt matters. In the voice-agent story from the posts, the context never reached the model, and only the actual prompt would show that.
2. **Sample** 2–10% of traffic, with higher rates for high-risk intents like pricing, legal and medical.
3. **Claim-level groundedness check:** split the answer into atomic claims, then verify each against the context, using an NLI model (cheap, fast) or a calibrated LLM judge. Hallucination rate = answers with at least one unsupported claim / answers checked. I also track unsupported claims per answer.
4. **Cheap 100% signals:**
   - **Citation verification:** does each cited span actually contain the stated fact? That's string or NLI matching, cheap enough to run on everything.
   - **Empty or low-score retrieval with a non-refusal answer.** If top similarity is below a threshold or zero chunks came back, but the model answered confidently, flag it. This one rule catches a lot.
   - Numbers and entities in the answer that don't appear in the context (regex or NER diff).
5. **Dashboards and alerts:** hallucination rate by intent, tenant and model version, alerting on a threshold or on significant week-over-week increases.
6. **Human audit** of flagged plus random samples weekly, to keep the detector calibrated and measure its precision and recall.

**Trade-offs:** an NLI check is fast (about 20–50 ms) but weaker at multi-sentence reasoning. An LLM judge is better but costs money and adds latency, so I run it async and offline, not inline. For high-risk flows, a **synchronous** groundedness check can block or rewrite the answer before it's sent.

Example: the 'answered despite empty retrieval' rule flagged 4% of calls within the first day. The root cause was a timeout that made retrieval return nothing, while the model answered from its priors anyway."

**Follow-ups:**
- *What about claims that are true but not in the context?* → By policy they're still ungrounded in RAG. Either allow general knowledge explicitly, or count it.
- *Inline or offline?* → Offline for monitoring. Inline only for high-stakes answers, where the added latency is worth paying.

**Red flags:**
- "We ask the model whether it hallucinated."
- Logging only the question and answer, without the context actually sent.
- No definition of what counts as a hallucination.

---

<a id="q14"></a>
## Q14. Given user feedback, how do you improve the system?
*Source: interview posts*

**What the interviewer is testing:** turning noisy feedback into diagnosed, measurable fixes.

**Strong answer:**

"User feedback is valuable but sparse and biased. Typically only 1–3% of users rate anything, and angry users rate more. So I treat it as a **triage signal**, not as a metric by itself.

**Loop:**
1. **Collect explicit and implicit signals:** thumbs up/down with an optional reason (wrong, outdated, incomplete, didn't understand), plus implicit signals like rephrasing, copy-paste of the answer (positive), escalation to a human, abandonment and citation clicks.
2. **Join feedback to the full trace:** query, retrieved chunks, prompt version, model and answer. Feedback without a trace can't be diagnosed.
3. **Cluster the negative feedback** by intent or topic (embedding clustering) and rank the clusters by volume × severity.
4. **Root-cause each cluster by stage:**
   - retrieval miss (the knowledge base lacks the doc, or chunking or indexing is wrong) → fix content or retrieval
   - retrieval hit but wrong answer → fix the prompt or model
   - correct but unhelpful (too long, too generic) → fix style or format
   - out of scope → better refusal or routing
   - **content gap** (the knowledge base doesn't cover it) → report to content owners. Often the highest-value fix.
5. **Convert confirmed failures into eval cases**, with a reference answer, in the regression suite.
6. **Fix, re-run the suite, A/B test**, and confirm the cluster's negative rate drops.

**Longer term:** preference pairs (a good vs bad answer to the same query) can feed DPO or reranker fine-tuning. Click data can train the retriever. I'd do that only after the cheap fixes are exhausted.

Example: the top negative cluster was 'leave policy for probation employees', at 22% thumbs-down. The trace showed retrieval returned the general leave policy, while the probation clause lived in a separate annexure with poor titles. Adding the annexure's title and section path to each of its chunks (contextual chunking) cut that cluster's thumbs-down rate to 6%."

**Follow-ups:**
- *How do you avoid overfitting to vocal users?* → Weight by segment, confirm with random-sample evaluation, and never tune only to thumbs-down.
- *What if feedback conflicts?* → Look at the user segments. The answer may need to be personalized or clarified, not changed.

**Red flags:**
- "We'll retrain the model on the feedback."
- Treating the thumbs-up rate as the quality metric.
- Not linking feedback to traces.

---

<a id="q15"></a>
## Q15. How do you build a ground-truth dataset, and where does human evaluation fit?
*Source: interview posts*

**What the interviewer is testing:** practical dataset construction: coverage, label quality and cost.

**Strong answer:**

"The golden set is the most valuable eval asset, so I build it deliberately.

**Sources, mixed:**
1. **Real production queries** (or pilot and search logs), sampled stratified by intent cluster. These reflect real phrasing, typos and ambiguity.
2. **Expert-written cases** for critical and rare scenarios: compliance, edge cases, questions the system must refuse on.
3. **Synthetic generation:** an LLM generates questions from a chunk, so the source chunk is the known gold evidence. It's cheap coverage for retrieval eval. Its weakness is that synthetic questions are often too easy and reuse the document's own wording. I prompt for paraphrase and multi-hop questions, and filter the result.
4. **Past incidents:** every production bug becomes a case.

**Each case contains:** the query, gold evidence (doc and span IDs), a reference answer or key facts, the expected behavior (answer, refuse, clarify, escalate), metadata tags (intent, difficulty, modality, tenant) and a 'last verified' date. That date matters because answers go stale when documents change.

**Label quality:**
- Written labeling guidelines with examples.
- Two labelers on a subset, measuring inter-annotator agreement (Cohen's kappa, above 0.7 as a target) and resolving disagreements, which usually reveal ambiguous guidelines.
- LLM-drafted references reviewed by an expert: about 3–5x faster than writing them from scratch.

**Where human evaluation fits:**
- Creating and validating the gold set.
- **Calibrating LLM judges** (see Q5).
- Periodic audits of production samples, and wherever judges are weak: domain nuance, tone, safety, multimodal content.
- High-stakes releases get a final human review.

**Sizing:** start with about 200 high-quality cases, then grow to 500–1,000 or more by merging in production failures. A few hundred carefully chosen cases beat 5,000 noisy synthetic ones.

Example: we started with 150 synthetic Q&A pairs and our metrics looked great, with Hit Rate@5 of 0.95. Once we added 200 real user queries, Hit Rate fell to 0.78. Real users say 'can I WFH from my hometown?' while the document says 'remote work location policy'. The synthetic set had been flattering the system."

**Follow-ups:**
- *How do you keep it fresh?* → Keep a 'last verified' date per case, re-verify cases whenever their source docs change (linked via doc IDs), and add a monthly slice of recent traffic.
- *Train/test contamination?* → Keep the golden set out of few-shot examples and fine-tuning data.

**Red flags:**
- Only synthetic data, generated by the same model being evaluated.
- No negative or refusal cases.
- No agreement check on the labels.

---

<a id="q16"></a>
## Q16. How do you evaluate agents: tool selection, the planner, and multi-agent systems at the agent, routing, orchestration and end-to-end levels? How do you use LLM-as-a-Judge for agentic workflows?
*Source: interview posts*

**What the interviewer is testing:** evaluating *trajectories*, not just final answers, and localizing failures in multi-step systems.

**Strong answer:**

"An agent can reach the right answer by luck, or reach the wrong answer after nine correct steps. So I evaluate both **the outcome and the trajectory**, at each level.

**1. Component / single-agent level:**
- **Tool selection accuracy:** for a labeled set of (state, expected tool), measure exact-match accuracy. Also measure 'no tool needed' precision, because over-calling tools is a real failure mode.
- **Argument correctness:** schema validity, and field-level match with the expected arguments.
- **Tool-result handling:** does the agent use the returned data correctly, and handle errors or empty results?
- Unit-test each agent in isolation with **mocked tools**, so the results are deterministic.

**2. Routing level** (supervisor or classifier picking an agent):
- A confusion matrix over routes, accuracy and per-route recall, plus the 'misroute cost', since some misroutes are harmless and others are dangerous.

**3. Planner / orchestration level:**
- **Plan quality:** does the plan cover all required subtasks with no unnecessary steps? Compare against reference plans, or judge with a rubric.
- **Trajectory metrics:** number of steps vs the optimal count, redundant calls, loops, recovery after a tool failure (inject faults deliberately), handoff correctness between agents, and whether state is passed correctly.
- A **trajectory-match** approach: compare the executed tool sequence to the reference, either strictly (exact order) or loosely (the required set of calls, in any order).

**4. End-to-end:**
- Task success rate on realistic scenarios, checked against the **final environment state**, not the text. For example: was the ticket actually created with the right fields? Was the refund amount correct? This is the most reliable signal.
- Cost, latency and step count per task, and the human-intervention rate.
- Safety: attempted disallowed actions, and whether HITL gates were respected.

**LLM-as-a-judge for agents:** give the judge the full trace (goal, each thought, tool call and observation) and a rubric with binary checks per step: 'Was this tool call justified by the previous observation? Did the agent act on stale data? Did it violate a constraint?' This localizes the first failing step. I calibrate it against human-annotated traces, and prefer programmatic state checks wherever they're possible.

**Nondeterminism:** run each scenario 3–5 times and report pass^k (all runs pass) as well as the mean. An agent that succeeds 70% of the time per run is unreliable.

Example: our support agent had 82% end-to-end success. Trajectory eval showed the router was fine at 97%, but the refund agent re-called `get_order` 3.4 times per task because it didn't trust the cached state. Fixing the state handoff cut average steps from 9 to 5 and raised success to 88%."

**Follow-ups:**
- *Where do reference trajectories come from?* → Expert demonstrations, successful production traces reviewed by humans, or scripted scenarios.
- *Is there a benchmark style for this?* → τ-bench-style simulated users with verifiable end-state checks are a good template.

**Red flags:**
- Only checking the final text answer.
- Running each scenario once and trusting the number.
- Evaluating against live tools with side effects instead of sandboxes or mocks.

---

<a id="q17"></a>
## Q17. Offline metrics improved but online CSAT didn't move. Why?
*[Added]*

**What the interviewer is testing:** that you understand the gap between offline evaluation and real user value.

**Strong answer:**

"I'd work through the hypotheses in order of likelihood:

1. **The offline set doesn't match production.** The golden set over-represents intents that were already fine, or is stale. If the improvement landed on 'table questions' and those are 3% of traffic, CSAT won't move. **Check:** re-weight the offline gains by the production intent mix.
2. **The metric doesn't measure what users care about.** We optimized faithfulness while users are unhappy about latency, verbosity, lack of actionable next steps, or needing to escalate. **Check:** read the negative-CSAT transcripts and tag their reasons.
3. **Judge gaming or bias** (see Q7). Answers got longer and the judge liked it. Users didn't. **Check:** human audit and output length trend.
4. **Measurement problems in the online test:** the experiment is underpowered (CSAT is noisy and sparse, and a 2-point lift may need tens of thousands of sessions), there's a bucketing bug, novelty effects, or the change didn't actually ship to the treatment arm. **Check:** confirm the exposure logs and run a power calculation.
5. **A downstream bottleneck:** answer quality improved, but CSAT is dominated by something else, such as a broken handoff to human agents, or the answer is correct but the policy itself frustrates users.
6. **Offsetting regressions:** latency rose 400 ms, or the refusal rate went up. That's why I track guardrail metrics beside the primary metric.

What I'd do: segment CSAT by intent and by whether the improved capability was even triggered. If the capability was used in only 5% of sessions, compare CSAT within those sessions. Then add online proxies closer to the change (rephrase rate, escalation rate on the affected intents), and update the offline set to reflect the real traffic mix.

Example: we raised correctness from 78% to 85% offline, but CSAT was flat. Segmenting showed that 60% of negative CSAT came from 'wanted a human, the bot kept trying'. The quality work was real but aimed at the wrong pain. Adding an early 'talk to an agent' option moved CSAT 6 points."

**Follow-ups:**
- *Which online metric would you trust more than CSAT?* → Task-level signals: resolution without escalation, repeat-contact within 7 days, rephrase rate.
- *Does that make offline eval useless?* → No. It's a necessary gate that prevents regressions. It just isn't sufficient proof of value.

**Red flags:**
- "Users are wrong; the metrics say it's better."
- Not checking statistical power of the online test.
- Never reading actual transcripts.

---

## Rapid-fire recap

- Evaluate retrieval, generation and system separately. End-to-end scores alone can't localize a failure.
- Recall@K is the ceiling for RAG. MRR/nDCG measure whether the best evidence is at the top.
- Faithfulness measures grounding, not truth. It can be 1.0 on outdated documents.
- Context recall and precision need references; faithfulness and answer relevancy don't, so they can run on production traffic.
- An LLM judge is a classifier: calibrate against humans (kappa) on a held-out set.
- Run pairwise judging in both orders, and test verbosity bias with padded answers.
- Separate the judge you optimize against from the judge you report with, to catch gaming.
- At p = 0.8, 200 questions gives ±5.5 points. Evaluate paired on the same questions and use bootstrap or McNemar.
- Every production incident becomes a regression test. Every shared-prompt change re-runs every dependent suite.
- Monitor hallucination with claim-level groundedness checks, plus a "confident answer despite empty retrieval" rule.
- Evaluate agents on trajectories and final environment state, run several times each (pass^k).
- An offline gain without an online gain usually means the eval set doesn't match production or the metric targets the wrong pain.
