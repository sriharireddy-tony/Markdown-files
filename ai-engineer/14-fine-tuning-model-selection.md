# 14 — Fine-Tuning, Model Selection & Optimization

**What interviewers probe here:** whether you can pick the *cheapest technique that solves the actual problem* (prompting → RAG → fine-tuning → training), and whether you understand what distillation, quantization and model routing cost you in quality. Candidates who answer "fine-tune it" by reflex score poorly. Candidates who explain behavior vs knowledge, data requirements and the eval that proves the change score well.

[← Back to index](README.md)

1. [RAG or fine-tuning — which is better? When do you choose each?](#q1)
2. [When are prompting and RAG not enough, so that fine-tuning is justified?](#q2)
3. [Explain LoRA/QLoRA, the data curation pipeline, and how you monitor overfitting vs generalization.](#q3)
4. [When would you use a smaller fine-tuned model instead of a frontier model API?](#q4)
5. [Explain distillation, quantization and pruning. When does each degrade quality unacceptably?](#q5)
6. [Open-weight vs closed models in a regulated industry.](#q6)
7. [Design a routing layer that picks between multiple models per request.](#q7)
8. [What is catastrophic forgetting, and how do you prevent it?](#q8)
9. [How much data do you need for fine-tuning, and when is synthetic data OK?](#q9)
10. [SFT vs DPO vs RLHF: which would you use for a behavior problem?](#q10)

---

<a id="q1"></a>
## Q1. RAG or fine-tuning: which is better? When do you choose each?
*Source: interview posts*

**What the interviewer is testing:** Do you know that the two techniques solve different problems, or do you treat it as a popularity contest?

**Strong answer:**

"I'd push back on the framing a little. They aren't competitors. **Fine-tuning changes how the model behaves. RAG changes what the model knows at request time.**

| Need | Right tool |
|---|---|
| Facts that change (prices, policies, inventory) | RAG |
| Facts that must be cited or audited | RAG |
| Per-customer or per-tenant knowledge | RAG, with one model and N knowledge bases |
| Consistent tone, format, persona | Prompting first, then fine-tuning |
| A narrow task done cheaply at high volume | Fine-tune a small model |
| Domain language or reasoning style the base model lacks | Fine-tuning (sometimes with RAG) |

A concrete example is a production voice agent serving many businesses in several Indian languages. Each business has its own prices and policies, and they change weekly. Fine-tuning would mean a training run per business, plus another every time someone edits a price. That's operationally impossible, and the model would still be stale between runs. With RAG it's one model and N knowledge bases: update a document and the agent is correct on the next call.

RAG isn't free, though. In that system the agent once invented eight facts in a row on a live call. Retrieval was fine, returning the right chunks at 0.85–0.9 similarity. The bugs were plumbing: the 'don't invent facts' rule was never attached to the prompt on the live path, and retrieval ran fire-and-forget, so the prompt was built before the results arrived. So most of the work in RAG isn't retrieval quality. It's making sure retrieved context actually reaches the model and that the model refuses when it doesn't have the answer.

Where I *would* fine-tune in that system: teaching the model to always answer in the caller's language, keep replies short for speech, and say 'let me connect you to sales' instead of guessing. That's behavior, not knowledge.

**My rule:** retrieve for facts, fine-tune for behavior, and often combine them. You can fine-tune a model to *use* retrieved context better, for example to cite and to refuse when context is missing."

**Follow-ups:**
- *Can fine-tuning inject knowledge?* → Somewhat, but it's unreliable, hard to update, and has no citations. Knowledge learned in fine-tuning also increases confident hallucination on nearby facts.
- *When do you use both?* → When you need domain behavior (format, reasoning style) *and* fresh facts, e.g. a medical summarizer fine-tuned for the note format that retrieves the patient's record.
- *What is RAFT?* → Fine-tuning on (question, retrieved docs including distractors, answer) so the model learns to use relevant context and ignore noise.

**Red flags:**
- "Fine-tuning is better because the model actually learns the data."
- Choosing without asking how often the data changes, or whether citations are needed.
- Not mentioning the cost of retraining or of evaluation.

---

<a id="q2"></a>
## Q2. When are prompting and RAG not enough, so that fine-tuning is justified?
*Source: interview posts*

**What the interviewer is testing:** Your escalation ladder, and whether you need evidence before spending on training.

**Strong answer:**

"I climb a ladder and only move up when I have eval evidence that the current rung has plateaued:

```
1. Better prompt (instructions, few-shot examples, output schema)
2. Better context (RAG, tools, better retrieval)
3. Bigger or better model
4. Fine-tune (usually a smaller model)
5. Continued pretraining / training from scratch (rare)
```

Fine-tuning is justified when one of these holds:

1. **Behavior won't stick through prompting.** The format, tone or policy is still violated in 5–10% of cases even with good few-shot examples.
2. **Cost or latency.** A frontier model works, but at 10M requests/month it's too expensive or too slow. You distill it into a small fine-tuned model.
3. **Prompt bloat.** You need 4k tokens of instructions and examples on every call. Fine-tuning bakes them in, so every request is shorter and cheaper.
4. **Narrow domain language.** Legal clauses, clinical codes, internal DSLs, or low-resource languages the base model handles poorly.
5. **Structured tasks at scale.** Classification, extraction or routing, where a 1–8B model fine-tuned on 5k examples can beat a frontier model zero-shot.
6. **Data can't leave your boundary**, so you have to self-host a smaller open model and adapt it.

Example: a ticket classifier with 40 categories. GPT-class zero-shot reached about 82% macro-F1 at around $0.004 per ticket. With 8k labeled historical tickets, a LoRA fine-tuned 8B model reached about 89% at about a tenth of the cost, and it ran inside our VPC. (Numbers are illustrative.) That's a clear win. For our Q&A assistant, fine-tuning would have added nothing, because the problem there was missing knowledge."

**Follow-ups:**
- *How do you know prompting has plateaued?* → Run a fixed eval set across prompt variants. If the last 3–4 iterations move the metric by less than a point and the error analysis shows the same failure type, you've plateaued.
- *What do you need before fine-tuning?* → A clean eval set, a baseline number, enough quality examples, and a retraining and ownership plan.

**Red flags:**
- Fine-tuning before trying few-shot prompting or structured output.
- Having no baseline to compare against.
- Ignoring the maintenance burden: every base-model upgrade means re-fine-tuning.

---

<a id="q3"></a>
## Q3. Explain LoRA/QLoRA, the data curation pipeline, and how you monitor overfitting vs generalization.
*Source: interview posts*

**What the interviewer is testing:** Real fine-tuning mechanics, not just library calls.

**Strong answer:**

"**LoRA:** instead of updating a full weight matrix W (d × k), you freeze it and learn a low-rank update ΔW = B·A, where B is d × r and A is r × k, with rank r around 8–64. Only A and B train. For a 4096×4096 projection that's 16.7M parameters frozen versus about 131k trainable at r = 16. Total trainable parameters are usually 0.1–1% of the model. You get small adapter files (tens of MB), cheap training, and you can hot-swap adapters per task or tenant on one base model.

**QLoRA:** load the frozen base in 4-bit (NF4) and train LoRA adapters in 16-bit on top, using double quantization and paged optimizers. A 7–8B model then fits for fine-tuning on a single 24 GB GPU, and a 70B on a single 80 GB card with some care. The trade-off is slightly slower training and a small quality gap compared with 16-bit LoRA.

Key hyperparameters are rank r, alpha (the scaling, often 2×r), target modules (attention projections only, or also MLP layers, which usually improves quality), learning rate (about 1e-4 to 2e-4 for LoRA), and epochs (1–3).

**Data curation pipeline:**
```
raw logs / docs / human labels
 → PII scrub → deduplicate (exact + near-dup via MinHash)
 → quality filter (heuristics + LLM judge + human spot-check)
 → format into instruction/chat template (same template as inference!)
 → balance classes / task types → decontaminate against eval set
 → train / val / test split (by user or time, not random rows, to avoid leakage)
```
Data quality beats quantity. 2k excellent examples usually beat 50k noisy ones.

**Overfitting vs generalization:**
- Watch training loss against validation loss. If val loss rises while train loss falls, stop or pick the earlier checkpoint.
- Evaluate on **task metrics**, not just loss: exact match, F1, judge scores on a held-out set.
- Run a **general capability regression** set (instruction following, safety, a slice of MMLU-style questions) to catch forgetting.
- Probe out-of-distribution inputs, such as paraphrases and new entities. A model that memorized will fail these.
- Signs of overfitting: it parrots training phrasing, gets worse on slightly different inputs, and its outputs become less diverse."

**Follow-ups:**
- *Which rank would you pick?* → Start at 16. Go up if the task is complex and underfits, down if you have little data.
- *Can you merge adapters?* → Yes. Merging into the base weights removes inference overhead, but you lose hot-swapping.
- *Why does the chat template matter?* → A mismatch between training and inference templates is a classic silent quality killer.

**Red flags:**
- "LoRA is a smaller model." (It's a low-rank update to a frozen model.)
- Splitting train and test randomly when near-duplicates exist.
- Judging success by training loss only.

---

<a id="q4"></a>
## Q4. When would you use a smaller fine-tuned model instead of a frontier model API?
*Source: interview posts*

**What the interviewer is testing:** Cost, latency and control trade-offs, backed by numbers.

**Strong answer:**

"I'd use a small fine-tuned model when the **task is narrow, the volume is high, and I can measure quality**. I'd keep the frontier API when the task is open-ended, low-volume or fast-changing.

| Factor | Small fine-tuned (1–8B) | Frontier API |
|---|---|---|
| Task breadth | Narrow: classify, extract, route, fixed format | Open-ended reasoning, long-tail questions |
| Volume | High (millions/month) | Low to medium |
| Latency | ~50–300 ms, self-hosted | ~0.5–5 s, variable |
| Cost at scale | Fixed GPU cost, low marginal cost | Linear in tokens |
| Data control | Stays in your VPC | Depends on vendor terms and region |
| Maintenance | You own training, serving, upgrades | Vendor owns it |
| Quality ceiling | High on the narrow task | Highest in general |

Illustrative break-even: an extraction task at 20M requests/month, 1,500 input + 200 output tokens each.
- Frontier API at roughly $3 per 1M input and $15 per 1M output tokens: 30B input tokens ≈ $90k, plus 4B output ≈ $60k, so about **$150k/month**.
- A fine-tuned 8B on 4 L4/A10-class GPUs, or 2 A100s with batching, is about **$5–10k/month** in infrastructure plus engineering time.

At that volume, fine-tuning pays for itself in weeks. At 50k requests/month it's about $400 on the API, and fine-tuning is pure overhead.

A common hybrid is **distillation plus a cascade**: the frontier model labels data to train the small model, the small model handles about 90% of traffic, and low-confidence cases escalate to the frontier model."

**Follow-ups:**
- *How do you get training data?* → Historical labeled data, or frontier-model outputs filtered by a judge and human review (check the vendor's terms on using outputs for training).
- *What's the hidden cost?* → Eval infrastructure, retraining when data drifts, GPU on-call, and re-tuning on base-model upgrades.

**Red flags:**
- Ignoring engineering and ops cost in the comparison.
- "Small models are always worse."
- Not mentioning a fallback path to the frontier model.

---

<a id="q5"></a>
## Q5. Explain distillation, quantization and pruning. When does each degrade quality unacceptably?
*Source: interview posts*

**What the interviewer is testing:** Whether you know the compression techniques and, more importantly, their failure modes and how to detect them.

**Strong answer:**

"All three trade some quality for cost and speed, in different ways.

**Distillation:** train a smaller student to mimic a larger teacher, either on the teacher's outputs (sequence-level, the common approach with LLMs) or on its logits and soft labels (richer signal, needs white-box access).
- *Pros:* big size reduction (70B → 8B), and it keeps task quality when the task is narrow.
- *Degrades when:* the task needs broad knowledge or long multi-step reasoning, when the student is too small for the task's capacity needs, or when the distillation data doesn't cover the long tail. The student looks great on the head of the distribution and falls apart on rare cases.

**Quantization:** store weights (and sometimes activations or the KV cache) in fewer bits.

Model memory math: params × bytes per param.
| Model | FP16 (2 B) | INT8 (1 B) | INT4 (0.5 B) |
|---|---|---|---|
| 8B | ~16 GB | ~8 GB | ~4–5 GB |
| 70B | ~140 GB | ~70 GB | ~35–40 GB |

(INT4 is a little above 0.5 B/param because of scales and zero-points. Add KV cache and activations on top.)

Methods: weight-only (GPTQ, AWQ, GGUF), weight + activation (SmoothQuant, FP8 on H100-class GPUs), and KV-cache quantization (FP8/INT8).
- *Pros:* 2–4× less memory, often faster decoding because decode is memory-bandwidth bound.
- *Degrades when:* you go below 4 bits; the model is small (a 3B model at INT4 hurts more than a 70B at INT4); the task is sensitive, such as math, code, long-context retrieval, structured JSON or non-English languages; or activation outliers are handled badly. Perplexity can look fine while exact-match tasks drop several points.

**Pruning:** remove weights (unstructured), or whole heads, channels or layers (structured).
- *Pros:* structured pruning gives real speedups. Unstructured pruning needs sparse-kernel support (e.g. 2:4 sparsity) to be faster.
- *Degrades when:* you prune aggressively without healing via fine-tuning or distillation. Degradation is often non-linear, collapsing past a threshold.

**How I decide 'unacceptable':** before compressing, I define a task eval with a threshold, for example 'no more than 1 point drop on extraction F1, zero JSON-validity regressions, no drop on the Hindi slice'. I compare per slice, not just the average. A typical failure is INT4 staying flat on English and losing about 6 points on Tamil."

**Follow-ups:**
- *Why is decoding faster with quantization?* → Decode is bandwidth-bound: fewer bytes per weight read per token means more tokens per second.
- *FP8 vs INT8?* → FP8 has more dynamic range, is natively supported on Hopper-class GPUs, and usually loses almost no quality.
- *Can you combine them?* → Yes. Distill, then quantize the student. Evaluate after each step.

**Red flags:**
- "Quantization has no quality loss."
- Evaluating only on perplexity or an aggregate score.
- Confusing distillation with fine-tuning on the same data.

---

<a id="q6"></a>
## Q6. Open-weight vs closed models in a regulated industry.
*Source: interview posts*

**What the interviewer is testing:** Whether you think about compliance, control and total cost, not just benchmarks.

**Strong answer:**

"In a regulated domain (banking, healthcare, insurance, government) I first ask: What data classification is involved? What do regulators require for data residency, auditability and explainability? Who is accountable if the vendor changes the model?

| Dimension | Open-weight (self-hosted) | Closed API |
|---|---|---|
| Data residency | Full control, on-prem or in-region VPC | Depends on vendor regions, contracts, zero-retention options |
| Version pinning | You freeze the exact weights forever | Vendor deprecates models; behavior can shift |
| Auditability | Weights, configs and logs are all yours | Limited to what the vendor exposes |
| Customization | Full fine-tuning, any method | Limited fine-tuning APIs |
| Quality | Strong, often behind the frontier on hard reasoning | Usually the best |
| Ops burden | GPUs, scaling, patching, security, on-call | Minimal |
| Licensing | Check the license (usage limits, acceptable-use clauses) | Vendor terms, DPA, BAA for HIPAA |
| Cost | Fixed, cheap at high volume | Variable, cheap at low volume |

Practical answer: many regulated companies go **hybrid**:
- PII-heavy or high-volume narrow tasks (claims extraction, KYC document parsing) → an open-weight model in their VPC.
- Low-sensitivity, hard reasoning tasks → a closed model through an enterprise agreement (zero data retention, regional endpoint, often via a cloud provider like Azure OpenAI, Bedrock or Vertex to inherit existing compliance).
- A gateway enforces the rule: 'data classified *restricted* can only route to the self-hosted model.'

Either way you need a model inventory, versioned prompts, evaluation evidence, human oversight for consequential decisions, and audit logs of inputs and outputs. That's where regimes like model-risk management and the EU AI Act for high-risk uses focus."

**Follow-ups:**
- *Can a closed API be compliant for PHI?* → Yes, with a BAA, the right region and zero retention, but legal and security have to sign off.
- *Biggest risk of closed models?* → Forced migration when a model is deprecated. You need an eval suite ready to requalify a new version.

**Red flags:**
- "Open source is always more secure."
- Ignoring licenses or vendor data-retention terms.
- Choosing purely on a leaderboard.

---

<a id="q7"></a>
## Q7. Design a routing layer that picks between multiple models per request.
*Source: interview posts*

**What the interviewer is testing:** System design for cost, quality and latency, plus how you'd prove the router works.

**Strong answer:**

"Goal: send each request to the cheapest model that will answer it well enough, within latency and compliance constraints.

```
request
  │
  ▼
[Policy filter] ── data class, tenant, region → allowed model set
  │
  ▼
[Router] ── rules + classifier (difficulty, intent, language, length)
  │
  ├─► small model (cheap, fast)  ──► [confidence / validator check] ──fail──┐
  ├─► mid model                                                             │
  └─► frontier model ◄──────────────────────────── escalate ────────────────┘
  │
  ▼
[Telemetry] cost, latency, quality signals → retrain router
```

**Routing strategies, simplest first:**
1. **Static rules:** by feature or endpoint ('summaries → small model, contract analysis → frontier').
2. **Classifier router:** a small model or embedding classifier predicts task type or difficulty. It's trained on logs labeled with which model's answer was acceptable (judged offline).
3. **Cascade:** try the small model first. If a validator fails (schema invalid, low log-prob confidence, judge score, or the model says 'unsure'), escalate. This gives the best cost savings but adds latency on escalated requests.
4. **Hard constraints:** context length (only some models take 200k tokens), language, tool-calling support, data residency.

**What makes it production-grade:**
- A single interface behind a gateway, with provider-specific prompt adapters (prompts don't transfer 1:1 between models).
- Fallbacks on errors or timeouts (see file 16).
- Per-route evals: the router is only as good as the offline 'which model was good enough' labels.
- Shadow mode first: route in the logs only, compare against always-frontier, then roll out.

**Illustrative result:** 70% of traffic to the small model, 25% to the mid model, 5% to the frontier model. Blended cost drops about 60%, with a quality drop under 1 point on the eval set and P95 latency improving because most requests hit the fast model."

**Follow-ups:**
- *Why not just use the LLM itself to route?* → That adds cost and latency to every request. A small classifier or rules are faster and deterministic.
- *How do you handle router mistakes?* → A cascade or validator catches under-routing. Periodic audits sample 'small-model answered' traffic and compare it with the frontier.
- *Is this like an LLM gateway?* → The router is a feature *inside* the gateway, alongside auth, rate limits and cost attribution.

**Red flags:**
- Routing without a way to measure quality per route.
- Ignoring that prompts must be adapted per model.
- No fallback when the chosen model fails.

---

<a id="q8"></a>
## Q8. What is catastrophic forgetting, and how do you prevent it?
*[Added]*

**What the interviewer is testing:** Awareness of fine-tuning side effects.

**Strong answer:**

"Catastrophic forgetting is when fine-tuning on a narrow task degrades capabilities the base model had: general instruction following, reasoning, safety refusals, other languages or formats. The gradient updates overwrite representations that weren't exercised by the new data.

**How I prevent and detect it:**
- **Parameter-efficient tuning (LoRA)** changes far fewer parameters, so forgetting is typically milder than with full fine-tuning (not zero).
- **Replay / data mixing:** mix 5–20% general instruction data, including safety examples, into the fine-tuning set.
- **Low learning rate, few epochs**, and early stopping on a *general* validation set, not just the task set.
- **Regression eval suite:** general benchmarks plus safety and refusal tests plus the product's other tasks, run before and after.
- **Keep the base available:** with adapters you can route non-task traffic to the untouched base model.

Example: we fine-tuned for a strict JSON extraction format, and the model then started answering chit-chat in JSON too. Adding 10% general chat data and routing only extraction traffic to the adapter fixed it."

**Follow-ups:**
- *Does LoRA eliminate forgetting?* → No, it reduces it. You still need a regression eval.
- *What about safety?* → Fine-tuning, even on benign data, can weaken refusals, so always rerun the safety eval.

**Red flags:**
- Never having heard of it, or assuming LoRA makes it impossible.

---

<a id="q9"></a>
## Q9. How much data do you need for fine-tuning, and when is synthetic data OK?
*[Added]*

**What the interviewer is testing:** Practical judgment about data.

**Strong answer:**

"It depends on the task. Illustrative ranges I'd start from:

| Goal | Typical examples |
|---|---|
| Style, format, tone | 100s to ~1k high-quality examples |
| Narrow classification / extraction | 1k–10k labeled examples |
| New domain behavior, complex reasoning | 10k–100k+ |
| Teaching a new language or knowledge | Usually continued pretraining; SFT is not enough |

I'd verify with a **learning curve**: train on 25%, 50% and 100% of the data. If quality is still rising steeply, get more data. If it's flat, data volume isn't the bottleneck; quality or labels are.

**Synthetic data is OK when:**
- It's generated by a stronger model and **filtered** (judge + rules + human spot-check of about 5%).
- It covers edge cases you lack real examples for (rare classes, adversarial inputs, long-tail formats).
- It's mixed with real data, and the eval set stays **real**, human-verified data.

**Risks:** reduced diversity (every example sounds the same), the teacher's errors being amplified, model collapse when you recursively train on your own outputs, vendor terms that restrict using outputs to train competing models, and contamination of the eval set."

**Follow-ups:**
- *How do you increase diversity?* → Vary personas, seeds, templates and difficulty; deduplicate by embedding; cap near-duplicates.
- *Label quality check?* → Measure inter-annotator agreement and audit a sample.

**Red flags:**
- "More data is always better."
- Evaluating on synthetic data generated by the same model.

---

<a id="q10"></a>
## Q10. SFT vs DPO vs RLHF: which would you use for a behavior problem?
*[Added]*

**What the interviewer is testing:** Understanding of post-training methods and when preference optimization helps.

**Strong answer:**

"**SFT (supervised fine-tuning):** train on (prompt → ideal response). Best when you can *write* the correct answer: format, style, task skill. It's the simplest, and the first step.

**RLHF (e.g. PPO):** train a reward model on human preference pairs, then optimize the policy against it with a KL penalty to stay near the SFT model. It's powerful but complex, unstable and compute-heavy: multiple models in memory, with a risk of reward hacking.

**DPO and relatives (IPO, KTO, ORPO):** optimize directly on preference pairs (chosen vs rejected) with a closed-form loss, with no separate reward model or RL loop. It's much simpler and more stable, and the default choice for most teams today.

**For a behavior problem:**
- 'Always return this schema / this tone' → **SFT**. The target is clear.
- 'Responses are correct but too verbose / too hedgy / occasionally unsafe', where the good behavior is easier to *judge* than to *write* → **DPO** on pairs (concise vs verbose, refuse vs comply). Pairs can come from user feedback (thumbs up/down), human raters or a judge.
- Full RLHF only when you have a large team, lots of preference data and need online optimization, or when a verifiable reward exists (RL on code tests or math answers).

The typical pipeline is SFT → DPO."

**Follow-ups:**
- *What's the risk with DPO?* → Over-optimization (e.g. learning 'shorter = better'). Watch length and keep a reference model via the beta parameter.
- *What does the KL penalty do?* → It keeps the policy from drifting too far and exploiting the reward model.

**Red flags:**
- Thinking RLHF is needed for every alignment task.
- Not knowing that DPO removes the explicit reward model.

---

## Rapid-fire recap

- Retrieve for facts, fine-tune for behavior. They combine; they don't compete.
- Climb the ladder (prompt → RAG → bigger model → fine-tune), and only move up with eval evidence.
- LoRA learns a low-rank ΔW = B·A on a frozen base. QLoRA adds a 4-bit NF4 base.
- Data quality beats quantity. Split by user or time, and decontaminate the eval set.
- A small fine-tuned model wins at high volume on narrow tasks. Compute the break-even.
- Model memory ≈ params × bytes (8B: ~16 GB FP16, ~4–5 GB INT4), plus KV cache.
- Quantization hurts small models, sub-4-bit precision, math/code/JSON and low-resource languages first. Evaluate per slice.
- Regulated industry: often hybrid, with a gateway enforcing data-class → model rules.
- Router: rules → classifier → cascade with a validator, rolled out in shadow mode first.
- Forgetting: replay data, low learning rate, a regression suite, and routing only task traffic to the adapter.
- Use SFT when you can write the answer, DPO when you can only judge it.
