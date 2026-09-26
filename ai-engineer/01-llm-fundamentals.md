# 01 — LLM Fundamentals

**What interviewers probe here:** whether you understand what happens *underneath* the API call. Candidates with 1–2 years of experience often can build RAG with LangChain but can't explain why embeddings work, what happens before the first token, or why output varies. These questions separate "framework users" from engineers who can debug.

[← Back to index](README.md)

1. [What happens before the first token](#q1)
2. [Attention mechanism](#q2)
3. [Tokenization vs embeddings](#q3)
4. [The "strawberry" problem](#q4)
5. [Tokens, parameters, context windows](#q5)
6. [Decoding strategies](#q6)
7. [Deterministic, consistent output](#q7)
8. [Repetition and mitigation](#q8)
9. [Why embeddings work](#q9)
10. [AI vs ML vs DL vs GenAI vs LLM vs Agent](#q10)
11. [Encoder vs decoder architectures](#q11)
12. [Positional encodings and long context](#q12)
13. [Pretraining vs instruction tuning vs RLHF/DPO](#q13)
14. [Why LLMs hallucinate](#q14)
15. [Reasoning models and test-time compute](#q15)

---

<a id="q1"></a>
## Q1. What happens between sending a prompt and the LLM generating its first token?
*Source: interview posts*

**What the interviewer is testing:** Whether you can trace the full inference path. This directly explains TTFT, cost and KV-cache behavior.

**Strong answer:**

"I'll split it into the request path and the model path.

**Request path:** the client sends the request. The API layer authenticates it, applies rate limits and applies the chat template (system/user/assistant turns wrapped in special tokens). Then the request is queued for a GPU worker. Under load, queueing is often the biggest slice of TTFT.

**Model path, the prefill phase:**
1. **Tokenization.** Text becomes token IDs using the model's BPE/SentencePiece vocabulary.
2. **Embedding lookup.** Each ID indexes a row in the embedding matrix, giving a vector of size d_model (e.g. 4096).
3. **Positional information.** Most modern models apply RoPE inside attention.
4. **Forward pass through N transformer layers.** Each layer does self-attention and then an MLP. All prompt tokens are processed **in parallel**, which makes prefill compute-bound.
5. **KV cache fill.** The Key and Value tensors for every prompt token at every layer are stored, so they are never recomputed.
6. **Logits.** The final hidden state of the *last* position is projected onto the vocabulary, softmaxed into probabilities, and **sampled** (temperature/top-p). That sample is the first token.

After that comes the **decode phase**: one token at a time, reusing the KV cache. Decode is memory-bandwidth-bound.

**Why it matters in production:** TTFT scales with prompt length. On one RAG service we sent about 8k tokens of context and TTFT was about 1.4 s. Cutting context to about 3k with a reranker, and putting the static system prompt first so prefix caching hit, brought it to about 450 ms."

**Follow-ups:**
- *Why is prefill compute-bound and decode memory-bound?* → Prefill does big parallel matrix multiplications over many tokens. Decode does one token per step but must read all the weights and the KV cache each step.
- *What is prefix caching?* → Reusing KV cache across requests that share an identical prompt prefix.
- *Where does a stop sequence apply?* → In decode, checked after each sampled token.

**Red flags:**
- "The model searches its memory for the answer."
- Not knowing that the prompt is processed in parallel but output is generated sequentially.
- Confusing tokenization with embedding.

---

<a id="q2"></a>
## Q2. Why is the attention mechanism important? Explain self-attention (Q/K/V, multi-head) simply.
*Source: interview posts*

**What the interviewer is testing:** Conceptual clarity, and whether you can connect attention to practical effects such as context cost and "lost in the middle".

**Strong answer:**

"Before transformers, RNNs compressed a whole sequence into one hidden state, passed step by step, so long-range information faded and training couldn't be parallelized. Attention lets every token **look directly at every other token** and decide how much each one matters.

Each token's vector is projected into three vectors:
- **Query**: what am I looking for?
- **Key**: what do I contain?
- **Value**: what do I pass on if selected?

Attention = softmax(QKᵀ / √d_k) · V. The dot product of Q and K scores relevance. Dividing by √d_k keeps the softmax from saturating. The output is a weighted mix of Values. In 'The bank raised rates, so it…', the token 'it' puts high weight on 'bank'.

**Multi-head** attention runs this, say, 32 times in parallel with different projections. One head can track syntax, another coreference, another position. In a decoder, a **causal mask** stops tokens from attending to future positions.

**Practical consequences I care about as an engineer:**
- Cost is O(n²) in sequence length for compute. The KV cache grows linearly, which is why long context is expensive.
- Attention isn't uniform. Models attend more strongly to the beginning and end of the context, and that's the 'lost in the middle' effect I test for in RAG.
- Variants such as GQA/MQA share K/V across heads to shrink the KV cache. That's a big serving win."

**Follow-ups:**
- *Why divide by √d_k?* → Large dot products push softmax into regions with tiny gradients. Scaling keeps variance around 1.
- *What is GQA?* → Groups of query heads share one K/V head, which cuts KV memory 4–8x with a small quality cost.
- *What does FlashAttention change?* → The same math, computed in tiles in SRAM to avoid materializing the n×n matrix. It's faster and uses less memory.

**Red flags:**
- Reciting the formula without explaining what Q/K/V mean intuitively.
- Saying attention "understands meaning".
- Not knowing the quadratic cost.

---

<a id="q3"></a>
## Q3. What is the difference between tokenization and embeddings?
*Source: interview posts*

**What the interviewer is testing:** A basic distinction that many candidates blur.

**Strong answer:**

"They're two different steps with different purposes.

| | Tokenization | Embedding |
|---|---|---|
| What | Splits text into sub-word units and maps them to integer IDs | Maps tokens or whole texts into dense float vectors |
| Output | `[3923, 374, 264]`: discrete, no meaning | `[0.12, -0.53, …]` of 768–3072 dims: continuous, semantic |
| Learned? | Vocabulary built by BPE/WordPiece from frequency statistics; a deterministic lookup | Learned weights (token embedding table) or a trained model (sentence embeddings) |
| Used for | Model input, counting cost and context limits | Similarity search, clustering, and as the model's internal representation |

There are two kinds of 'embedding' worth separating:
1. **Token embeddings** are the first layer inside the LLM, one vector per token ID.
2. **Sentence/document embeddings** come from a separate embedding model (e.g. BGE, text-embedding-3). It pools a whole passage into one vector, and that's what goes into a vector DB for RAG.

So the pipeline is text → tokenizer → IDs → embedding layer → vectors → transformer. In RAG, chunk text → embedding model tokenizes internally → one vector per chunk."

**Follow-ups:**
- *Is token count the same across models?* → No. Each model has its own tokenizer, so the same text can differ by 10–30% in tokens, and Indic/CJK languages often cost 2–4x more tokens.
- *Can you use an LLM's token embeddings for search?* → Not well. They're per-token and not trained for sentence similarity.

**Red flags:**
- "Tokenization converts text to vectors."
- Assuming one token equals one word.

---

<a id="q4"></a>
## Q4. Prove why GPT can't reliably count the r's in "strawberry".
*Source: interview posts (30-day build roadmap)*

**What the interviewer is testing:** Whether you understand that models see tokens, not characters, and can demonstrate it hands-on.

**Strong answer:**

"The model never sees letters. It sees token IDs. 'strawberry' is split into two or three sub-word tokens, and the model has to have *memorized* how each token is spelled to count characters. Nothing in the architecture gives it character-level access."

```python
import tiktoken
enc = tiktoken.get_encoding("cl100k_base")
ids = enc.encode("strawberry")
print(ids)                                   # a few integer IDs, not 10 characters
print([enc.decode([i]) for i in ids])        # e.g. ['str', 'aw', 'berry']
print(len("strawberry"), "chars vs", len(ids), "tokens")

# Compare spacing and casing; the tokenization changes
for w in ["strawberry", " strawberry", "Strawberry", "s t r a w b e r r y"]:
    print(repr(w), [enc.decode([i]) for i in enc.encode(w)])
```

"Once you print it, the failure is obvious. The two r's in 'berry' live inside one token, so counting requires internal knowledge that token 'berry' contains 'rr'. That knowledge is weak and inconsistent. If I space out the letters, each letter becomes its own token and accuracy jumps.

**Engineering lesson:** don't ask an LLM to do character-level or exact arithmetic work. Give it a tool (`len`, a Python sandbox, a regex). The same root cause explains failures at reversing strings, rhyming, exact word counts and some multi-digit arithmetic. Newer reasoning models can often get 'strawberry' right by spelling it out step by step, but that's a workaround, not character access."

**Follow-ups:**
- *Why does it also struggle with math like 123456 × 789?* → Numbers get chunked into arbitrary multi-digit tokens. Use a calculator tool.
- *How would you fix it in a product?* → Route it to deterministic code through tool calling.

**Red flags:**
- "The model is just dumb" or "it needs more training data."
- Not being able to show the tokenization.

---

<a id="q5"></a>
## Q5. What do tokens, parameters and context windows mean in practice for cost, latency and limits?
*Source: interview posts*

**What the interviewer is testing:** Translating concepts into engineering and business consequences.

**Strong answer:**

- **Tokens** are the unit of billing and compute. Roughly 1 token ≈ 0.75 English words. Output tokens usually cost 3–5x more than input tokens and dominate latency, because they're generated one at a time.
- **Parameters** are the learned weights. More parameters means more capability, but also more GPU memory (about 2 bytes per parameter in FP16, so a 70B model needs about 140 GB just for weights) and slower per-token decode.
- **Context window** is the maximum input plus output tokens per request (e.g. 128k–1M). A bigger window doesn't mean better use: cost and TTFT rise with context, and recall of details in the middle drops.

"A quick cost model I use: 50k requests/day × (3k input + 400 output) tokens. At $1 per M input and $4 per M output, that's 150M × $1 + 20M × $4 = $150 + $80 = **$230/day**. That tells me the fastest savings are reducing input context (a reranker, trimming chat history) and caching the static prefix. The output is already small."

**Follow-ups:**
- *Why is output more expensive?* → Sequential decode ties up the GPU per token. Input is prefilled in parallel.
- *What happens when you exceed the window?* → The API errors, or the framework silently truncates. Always count tokens before sending.

**Red flags:**
- "Just use the 1M context and put everything in."
- Not being able to estimate a cost.

---

<a id="q6"></a>
## Q6. Explain decoding strategies: greedy, temperature, top-k, top-p, beam search.
*Source: interview posts (roadmap)*

**What the interviewer is testing:** Control over output behavior, and knowing which settings fit which task.

**Strong answer:**

"At each step the model outputs a probability distribution over the vocabulary. Decoding decides how to pick from it.

| Strategy | How | Use when |
|---|---|---|
| Greedy | Always take the argmax | Extraction, classification; can loop or repeat |
| Temperature | Divide logits by T before softmax. T<1 sharpens, T>1 flattens | 0–0.3 for factual/structured output, 0.7–1.0 for creative work |
| Top-k | Sample only from the k most likely tokens | Cuts the long tail of nonsense |
| Top-p (nucleus) | Sample from the smallest set whose cumulative probability ≥ p | Adaptive; usually preferred over top-k |
| Beam search | Keep the b best partial sequences | Translation/ASR; rarely used for chat LLMs (bland, repetitive, expensive) |

Settings I'd actually use: RAG Q&A at temperature 0–0.2 with top-p 1. JSON extraction at temperature 0 plus constrained decoding. Brainstorming at 0.8–1.0 with top-p 0.95. I usually tune temperature *or* top-p, not both at once, so I can see which one caused a change.

Newer reasoning models sometimes fix or restrict sampling parameters, so I check provider docs rather than assuming."

**Follow-ups:**
- *Does temperature 0 guarantee identical output?* → No; see Q7.
- *What is min-p?* → Keep tokens whose probability ≥ min_p × p(top token). It adapts to the model's confidence.

**Red flags:**
- "Temperature controls creativity" with no mechanism.
- Recommending beam search for chat.

---

<a id="q7"></a>
## Q7. An LLM gives different answers to the same prompt. How do you make the output deterministic and consistent?
*Source: interview posts (MNC GenAI interview)*

**What the interviewer is testing:** Knowing that true determinism is hard, and that consistency is an engineering problem with several layers.

**Strong answer:**

"First I'd separate **bit-exact determinism** from **business consistency**. Users usually need the second: the same facts and the same decision, even if the wording differs.

**Sources of variation:**
1. Sampling (temperature/top-p).
2. Even at temperature 0, GPU floating-point non-associativity and **batching**. Your request is batched with different requests each time, which changes kernel reduction order, so near-tie logits can flip.
3. The provider silently updating the model behind an alias.
4. Non-deterministic *inputs*: retrieval returning chunks in a different order, timestamps in the prompt, chat history differences.

**What I do, in layers:**
- Temperature 0 and a fixed `seed` where the API supports it (it's best-effort).
- **Pin the model version** (a dated snapshot, not 'latest').
- Make inputs deterministic: stable sort of retrieved chunks by (score, doc_id), no volatile fields in the prompt.
- **Constrain the output**: structured output/JSON schema or an enum of allowed labels, so there's less room to drift.
- **Cache** answers for identical normalized inputs. The same question returns the same stored answer, which is the only true guarantee.
- For high-stakes decisions, self-consistency: sample 3–5 times and take a majority vote, or push the decision to deterministic code and let the LLM only extract fields.
- Measure it: I run the same 200 eval prompts 5 times and track the **agreement rate**. On one classification service that went from 91% to 99.4% after we added enum constraints and sorted retrieval."

**Follow-ups:**
- *Why does batching affect results?* → Different batch sizes lead to different kernel paths and reduction orders, which cause tiny numeric differences that flip argmax on near-ties.
- *Self-hosted?* → Batch-invariant kernels or batch size 1 give true determinism at a throughput cost.

**Red flags:**
- "Just set temperature to 0," end of answer.
- Not mentioning model version pinning or retrieval ordering.

---

<a id="q8"></a>
## Q8. Why do LLMs sometimes repeat the same sequence, and how do you mitigate it?
*Source: interview posts*

**What the interviewer is testing:** The mechanism of degeneration and practical fixes.

**Strong answer:**

"**Why it happens:** decoding is self-reinforcing. Once a phrase appears, attention to it raises the probability of continuing the same pattern, and greedy/low-temperature decoding keeps picking the argmax, so you get a loop. It's worse with small or heavily quantized models, very long generations, a context full of repetitive content (e.g. repeated table rows in retrieved chunks), and a missing or ignored stop token.

**Mitigations:**
- **Sampling:** a slightly higher temperature (0.3–0.7) or top-p instead of pure greedy decoding.
- **Penalties:** `repetition_penalty` (e.g. 1.1–1.2 in vLLM/HF), or `frequency_penalty`/`presence_penalty` in OpenAI-style APIs. Don't overdo them, because they damage code and JSON where repeated tokens are legitimate.
- `no_repeat_ngram_size` for some seq2seq tasks.
- **Proper stopping:** correct chat template and EOS token (a very common cause with self-hosted models), stop sequences, and a sensible `max_tokens`.
- **Fix the input:** dedupe retrieved chunks and don't feed the model its own previous long outputs verbatim.
- **Detect it in post-processing:** an n-gram repetition detector that truncates and retries.

Real case: a self-hosted fine-tuned model looped 'Thank you for your query' until max tokens. The root cause was that we used the base chat template, so the model never emitted the EOS token it was trained with. Fixing the template solved it without any penalty tuning."

**Follow-ups:**
- *Frequency vs presence penalty?* → Frequency scales with how many times a token appeared. Presence is a flat penalty once it has appeared at all.

**Red flags:**
- Only saying "increase temperature".
- Not considering template/EOS issues on self-hosted models.

---

<a id="q9"></a>
## Q9. Why do embeddings work at all?
*Source: interview posts*

**What the interviewer is testing:** Understanding beyond "vectors represent meaning".

**Strong answer:**

"Two ideas.

1. **Distributional hypothesis:** words that appear in similar contexts have similar meanings. If a model is trained to predict context, the internal representations of 'doctor' and 'physician' end up close together because they're interchangeable in the training signal.

2. **Training objective shapes the geometry.** Sentence embedding models are trained **contrastively**: pull (query, relevant passage) pairs together and push the query away from random or hard-negative passages (InfoNCE loss). After millions of pairs, distance in the space means *relevance as defined by the training data*.

That second point has practical consequences:
- 'Similar' means whatever the training pairs taught. A general model may put 'Python the language' and 'python the snake' closer than you'd like, or treat 'refund allowed' and 'refund not allowed' as near-identical because negation is a weak signal.
- Embeddings are **lossy compression**. Exact identifiers like SKU-4471 or error codes don't embed well, which is why I add BM25.
- Domain shift hurts. On legal or medical text, fine-tuning on in-domain pairs can raise recall@10 by 10–20 points."

**Follow-ups:**
- *What are hard negatives?* → Passages that look relevant but aren't. They teach fine distinctions.
- *Why cosine similarity?* → Normalized vectors make direction the signal and remove length effects.

**Red flags:**
- "The model understands the meaning."
- Not knowing embeddings are trained for a specific objective.

---

<a id="q10"></a>
## Q10. AI vs ML vs DL vs GenAI vs LLM vs Agent vs Agentic AI: give crisp distinctions.
*Source: interview posts (roadmaps)*

**What the interviewer is testing:** Clear vocabulary, and no hype.

**Strong answer:**

| Term | Crisp definition |
|---|---|
| AI | Any system performing tasks that normally need human intelligence (includes rule-based systems) |
| ML | Systems that learn patterns from data instead of explicit rules |
| Deep Learning | ML using multi-layer neural networks |
| Generative AI | Models that generate new content (text, images, audio) |
| LLM | A large transformer trained on text to predict the next token; one kind of GenAI |
| AI Agent | An LLM in a loop that can **decide** actions, call tools, observe results and continue until a goal is met |
| Agentic AI | The broader system property: autonomy, planning, memory, tools and multi-step or multi-agent orchestration |

"The key line is between a **workflow** and an **agent**. If the code decides the control flow, it's a workflow with LLM steps. If the model decides the next step, it's an agent. Most production value today comes from workflows with small agentic pockets."

**Follow-ups:**
- *Is a RAG chatbot an agent?* → Not by default. Fixed retrieve-then-generate is a pipeline. It becomes agentic when the model decides whether, what and how many times to retrieve.

**Red flags:**
- Calling every chatbot an "agent".

---

<a id="q11"></a>
## Q11. Encoder-only vs decoder-only vs encoder-decoder. Why are most LLMs decoder-only?
*[Added]*

**What the interviewer is testing:** Architectural awareness and picking the right model for the task.

**Strong answer:**

| Type | Attention | Examples | Best for |
|---|---|---|---|
| Encoder-only | Bidirectional | BERT, RoBERTa, BGE/E5 embedders, cross-encoder rerankers | Classification, embeddings, reranking, NER |
| Decoder-only | Causal (left to right) | GPT, Llama, Claude, Gemini | Open-ended generation, chat, reasoning |
| Encoder-decoder | Bidirectional encoder + causal decoder | T5, BART, Whisper | Translation, summarization, ASR |

"Decoder-only won for general LLMs because the next-token objective uses *every* token of raw text as a training signal, it scales simply, and one model covers every task through prompting. KV caching is also natural in causal models.

In practice I still use encoder models heavily: a BGE embedder and a cross-encoder reranker in RAG, and a fine-tuned DeBERTa for intent classification at about 5 ms and near-zero cost instead of an LLM call."

**Follow-ups:**
- *Why is a cross-encoder reranker encoder-only?* → It needs bidirectional attention over the query and passage together to output a relevance score.

**Red flags:**
- Thinking an LLM is the right tool for every NLP task.

---

<a id="q12"></a>
## Q12. How do positional encodings (RoPE) work, and why does long context degrade?
*[Added]*

**What the interviewer is testing:** Depth on long context. It directly affects RAG design.

**Strong answer:**

"Attention by itself is order-agnostic, so the model needs position information. The original transformer added sinusoidal vectors to embeddings. Most modern LLMs use **RoPE** (rotary position embeddings). They rotate the Q and K vectors by an angle proportional to position, so the dot product between two tokens depends on their *relative* distance. That generalizes better and allows context extension tricks such as position interpolation, NTK scaling and YaRN.

**Why long context degrades even inside the advertised window:**
- The model saw far fewer training examples at extreme lengths, so extended positions are undertrained.
- Attention gets diluted: with 200k tokens, the relevant 200 tokens compete with a lot of noise.
- Primacy/recency bias gives the U-shaped 'lost in the middle' curve.
- Multi-fact reasoning (combining facts spread across the context) degrades much faster than single-fact 'needle in a haystack' retrieval.

**Engineering takeaway:** I treat the effective context as much smaller than the advertised one. I retrieve and rerank to the top 5–10 chunks, put the most relevant first or last, and benchmark my own task at 8k/32k/128k before relying on long context."

**Follow-ups:**
- *Needle-in-a-haystack passes. Is that enough?* → No. It tests single-fact lookup. Test multi-hop and aggregation too.

**Red flags:**
- "The model supports 1M tokens, so recall is perfect."

---

<a id="q13"></a>
## Q13. Pretraining vs instruction tuning vs RLHF/DPO: what does each stage give the model?
*[Added]*

**What the interviewer is testing:** Knowing where capabilities and behaviors come from. It informs fine-tuning decisions.

**Strong answer:**

| Stage | Data | Gives the model |
|---|---|---|
| Pretraining | Trillions of tokens of raw text, next-token prediction | Knowledge, language and reasoning ability; a "document completer" |
| Instruction tuning (SFT) | Tens of thousands to millions of (instruction, ideal response) pairs | Follows instructions, chat format, tool-call format |
| Preference tuning (RLHF with a reward model + PPO, or DPO directly on preference pairs) | Human/AI rankings of responses | Helpfulness, harmlessness, tone, refusals; reduces bad behaviors |

"Knowledge mostly comes from pretraining. SFT and preference tuning shape **behavior**. That's why I say fine-tuning is poor at injecting new facts but good at format, style and task specialization. For knowledge, use RAG.

RLHF also explains some quirks: models can be over-confident or sycophantic because raters preferred confident, agreeable answers."

**Follow-ups:**
- *DPO vs PPO?* → DPO skips the separate reward model and the RL loop. It's simpler and more stable, and it's the common choice for teams fine-tuning open models.

**Red flags:**
- "Fine-tune the model on our documents so it knows them."

---

<a id="q14"></a>
## Q14. Why do LLMs hallucinate at a fundamental level?
*[Added]*

**What the interviewer is testing:** Understanding root causes, so your mitigations aren't cargo cult.

**Strong answer:**

"An LLM is trained to produce **plausible continuations**, not true ones. There's no built-in 'I don't know' signal. Specific causes:
1. **Objective mismatch.** Next-token prediction rewards fluency. A confident wrong answer and a right answer can have similar likelihood.
2. **Knowledge gaps and compression.** Rare facts (long-tail entities, exact numbers) are stored weakly, so the model reconstructs something plausible.
3. **Training and eval incentives.** Benchmarks and preference data often reward answering over abstaining, so guessing is learned.
4. **Decoding.** Sampling can pick a low-probability wrong token, and then the model stays consistent with its own error.
5. **Context problems** (the most common in production): missing, irrelevant or conflicting retrieved context, or the prompt never actually containing the context.

**Mitigations map to causes:** grounding via RAG with 'answer only from context', an explicit abstain path, lower temperature, citations with verification, and output checks (NLI or LLM-judge faithfulness). In my experience most 'hallucinations' in RAG apps turned out to be pipeline bugs: context not attached, truncated or wrong. So I debug the plumbing before blaming the model."

**Follow-ups:**
- *Can you eliminate hallucination?* → No. You reduce it, detect it and design safe failure modes (refuse or escalate).

**Red flags:**
- "Fine-tuning fixes hallucination."
- Only blaming the model.

---

<a id="q15"></a>
## Q15. Reasoning models and chain-of-thought: when is test-time compute worth the cost?
*[Added]*

**What the interviewer is testing:** Cost/latency/quality judgment with modern models.

**Strong answer:**

"Reasoning models (and CoT prompting) spend extra tokens 'thinking' before answering. That helps on problems with multiple dependent steps: math, complex code, multi-constraint planning, analyzing long documents, and agent planning. It adds little for lookup, classification, extraction or simple RAG answers.

Trade-offs:
- **Latency:** thinking can add seconds to tens of seconds, which is unacceptable for voice or autocomplete.
- **Cost:** thinking tokens are billed, often 3–10x the visible output.
- **Variance:** harder to make deterministic.

My approach: **route by difficulty**. A cheap classifier or the fast model handles most traffic. Hard cases (low-confidence classification, multi-step requests, failed validation) escalate to the reasoning model or a higher thinking budget. On one internal analytics assistant, about 15% of queries went to the reasoning tier. That kept average cost around 1.8x the fast model while matching the all-reasoning quality on our eval set within 2 points."

**Follow-ups:**
- *Should you show the chain of thought to users?* → Usually not raw. Show a summarized rationale and citations.

**Red flags:**
- "Always use the smartest model."

---

## Rapid-fire recap
- Prefill processes the prompt in parallel and fills the KV cache. Decode generates tokens one at a time.
- TTFT grows with prompt length and queueing. Output tokens drive total latency and cost.
- Attention is softmax(QKᵀ/√d)V: every token weighs every other token, at O(n²) cost.
- Tokenization gives discrete IDs. Embeddings give dense semantic vectors.
- "Strawberry" fails because the model sees sub-word tokens, not characters. Use tools for character or math work.
- Temperature 0 isn't bit-exact: batching and floating-point effects plus model updates. Pin versions, constrain outputs, cache.
- Repetition comes from self-reinforcing decoding. Fix it with penalties, sampling, and a correct EOS/template.
- Embeddings encode whatever similarity the contrastive training taught.
- Effective context is smaller than the advertised window. Test your own task.
- Knowledge comes from pretraining. SFT and preference tuning shape behavior.
- Most RAG hallucinations are plumbing failures. Verify the context actually reached the model.
- Route to reasoning models only for multi-step problems.
