# 11 — Memory & Context Management

**What interviewers probe here:** whether you treat memory as a data-engineering problem (what to write, when, where, and how to expire it) rather than "append everything to the chat history". Expect questions about cost, context-window limits, stale or conflicting memories, and privacy.

[← Back to index](README.md)

1. [Short-term vs long-term memory; working state vs semantic vs episodic memory](#q1-short-term-vs-long-term-memory-working-state-vs-semantic-vs-episodic-memory)
2. [Types of memory management in RAG/chat systems and best practices](#q2-what-types-of-memory-management-exist-in-rag-and-chat-systems-and-what-are-the-best-practices)
3. [What to remember vs deliberately forget](#q3-what-should-an-agent-remember-and-what-should-it-deliberately-forget)
4. [Preventing unbounded memory growth](#q4-how-do-you-stop-memory-growing-unbounded-across-a-long-session)
5. [Long conversation history and context-window limits](#q5-how-do-you-handle-long-conversation-history-and-context-window-limits)
6. [When to summarize vs store in full](#q6-when-do-you-summarize-past-context-instead-of-storing-it-in-full)
7. ["Summarization costs money too, isn't that counterproductive?"](#q7-summarizing-history-with-an-llm-costs-money-too-isnt-that-counterproductive)
8. [Architecting real-time conversation summarization](#q8-how-would-you-architect-real-time-conversation-summarization)
9. [Maintaining context in a long-running chatbot](#q9-how-do-you-maintain-context-in-a-long-running-chatbot)
10. [Conflicting information in memory](#q10-how-do-you-handle-conflicting-information-in-memory)
11. [Challenges of memory in long-running agentic systems](#q11-what-are-the-common-challenges-of-memory-in-long-running-agentic-systems)
12. ["Forget everything about me": privacy and GDPR](#q12-a-user-says-forget-everything-about-me-how-does-your-memory-design-support-that-privacygdpr)

---

## Q1. Short-term vs long-term memory; working state vs semantic vs episodic memory
*Source: interview posts*

**What the interviewer is testing:** Whether you can map memory types to concrete storage and lifetimes, not just recite definitions.

**Strong answer:**

The LLM itself is stateless. Every kind of "memory" is something my application stores and chooses to re-inject into the context. I split it by lifetime and by what it contains:

| Type | What it holds | Lifetime | Typical store |
|---|---|---|---|
| **Short-term / working state** | Current messages, tool results, plan, scratchpad variables | One task or session | In-process state, a LangGraph checkpoint, Redis |
| **Semantic (long-term)** | Durable facts: "user prefers Python", "company fiscal year starts in April" | Months | Vector DB and/or a key-value profile table |
| **Episodic (long-term)** | Records of past interactions: "last Tuesday we debugged the payroll export and the fix was X" | Weeks to months | Vector DB of episode summaries with timestamps |
| **Procedural** | How to do things: learned instructions, few-shot examples, refined prompts | Versioned | Prompt registry, config |

Short-term memory answers "what are we doing right now?" Long-term memory answers "what do I know about this user or world that should shape this conversation?"

In practice, working state is structured (a typed state object with `messages`, `plan`, `retrieved_docs`, `step_count`), not only a chat log. Semantic memory is best stored as **structured facts with keys**, e.g. `user.language = "Telugu"`, because keyed facts can be updated and deleted. Episodic memory is stored as **summaries with metadata** (time, topic, outcome) and retrieved by similarity plus recency.

Example: in an HR assistant, the working state holds the current leave-request draft. Semantic memory holds "employee is in the Hyderabad office, uses the 2025 leave policy". Episodic memory holds "last month this employee's reimbursement was rejected for a missing receipt", so the assistant proactively asks for the receipt this time.

**Follow-ups:**
- *Is the context window itself memory?* → It's working memory only. It's bounded, expensive, and lost when the request ends.
- *Where does RAG fit?* → RAG is shared, organization-level knowledge retrieval. Long-term memory is per-user or per-agent and written by the system itself.
- *Why not put everything in the vector DB?* → Facts like preferences need exact update and delete. Similarity search can't guarantee you overwrite the old value.

**Red flags:**
- "The model remembers previous conversations." (It doesn't, the app does.)
- Treating memory as just `messages.append()` with no expiry or structure.
- No distinction between per-user memory and shared knowledge.

---

## Q2. What types of memory management exist in RAG and chat systems, and what are the best practices?
*Source: interview posts*

**What the interviewer is testing:** Knowledge of the concrete strategies and the trade-off each one makes.

**Strong answer:**

The common strategies, from simplest to most sophisticated:

1. **Full buffer.** Send the entire history. It's perfect recall, but cost grows linearly per turn, so total cost grows quadratically over a conversation. Only fine for short sessions.
2. **Sliding window.** Keep the last N turns or last K tokens. Cheap and predictable, but it forgets the user's goal stated in turn 1.
3. **Summary memory.** Rolling summary of old turns plus the last few verbatim turns. A good default.
4. **Summary + window hybrid.** `[system] [running summary] [last 6 turns] [retrieved context] [question]`. This is what I usually ship.
5. **Vector-retrieved memory.** Embed past turns or episodes and retrieve only the relevant ones for the current query. Scales to very long histories.
6. **Entity / structured memory.** Extract facts (name, account ID, preferences, open issues) into a profile or state object. This is the most reliable way to carry key facts forward.

Best practices I follow:
- **Separate retrieval context from conversation memory.** RAG chunks are per-turn and should not accumulate in history. Re-retrieve each turn instead of carrying old chunks along.
- **Token budget per section.** For example: system 1.5k, memory summary 800, recent turns 2k, retrieved docs 4k, answer reserve 1k. Enforce it in code.
- **Condense the query.** Rewrite follow-ups ("what about contractors?") into standalone questions before retrieval.
- **Pin critical facts.** Things like the user's goal, constraints and IDs live in structured state that's never summarized away.
- **Scope memory by user and tenant.** Memory is a data-leak surface.
- **Evaluate memory.** Build tests like "fact stated in turn 2, asked about in turn 30".

**Follow-ups:**
- *Why not carry previously retrieved chunks forward?* → They bloat context and go stale. Re-retrieving with the condensed query is cheaper and more relevant.
- *Which strategy for a support bot averaging 8 turns?* → Window plus entity memory. Summarization is overkill at that length.

**Red flags:**
- Naming LangChain memory classes without explaining the trade-offs.
- No token budget. "The context is 128k, so it's fine."

---

## Q3. What should an agent remember, and what should it deliberately forget?
*Source: interview posts*

**What the interviewer is testing:** Judgment about memory write policies: signal vs noise, privacy, and staleness.

**Strong answer:**

I treat memory writes like database writes: there's an explicit policy, not "save everything".

**Remember** (high value, stable, reusable):
- User preferences and constraints: language, format, timezone, role.
- Durable facts the user confirmed: "my manager is Priya", "we use Postgres 15".
- Outcomes of tasks: decisions made, what worked, what failed and why (episodic).
- Open commitments: "follow up on ticket 4412 on Friday".

**Forget or never store:**
- Raw tool outputs and intermediate reasoning. Keep a summary or a pointer to the log instead.
- Sensitive data with no ongoing purpose: card numbers, OTPs, health details, passwords. Redact before writing.
- Transient states: "user is on the checkout page".
- Low-confidence inferences. "User seems frustrated" shouldn't become a permanent fact.
- Anything the user corrects or retracts. It gets deleted, not just outranked.

How I implement it:
- A **memory extractor** step at the end of a session or task. A small, cheap model outputs candidate facts with a type, a confidence score and a TTL.
- A **write gate**: dedupe against existing memory, reject low-confidence items and run PII redaction.
- **TTLs by type**: preferences never expire (until changed), task context expires after 30 days, episodic summaries expire after 90 days unless accessed.
- **Importance × recency × access frequency** scoring to decide what gets evicted.

Example: a coding assistant remembers "repo uses pnpm and Vitest" (saves future mistakes) but not the 400-line stack trace from yesterday. It stores only "the flaky auth test was caused by a timezone assumption, fixed in commit abc123".

**Follow-ups:**
- *Who decides, the LLM or code?* → The LLM proposes candidates, and code enforces policy (PII, TTL, dedupe, limits).
- *How do you know memory helps?* → A/B test memory on vs off. Measure task success and turns-to-resolution, and track "memory was wrong" feedback.

**Red flags:**
- "Store everything, retrieval will sort it out."
- No mention of PII or of user corrections.

---

## Q4. How do you stop memory growing unbounded across a long session?
*Source: interview posts*

**What the interviewer is testing:** Concrete bounding mechanisms and their costs.

**Strong answer:**

Unbounded memory causes three failures: cost grows every turn, latency grows, and quality drops because relevant facts get buried ("lost in the middle"). So I put a hard budget on every layer:

1. **Hard token budget on working context.** Something like 8k for history. When it's exceeded, trigger compaction. Never silently truncate from the front, because that drops the user's original goal.
2. **Compaction:** summarize the oldest turns into the running summary. Keep the last N turns verbatim and keep pinned facts in structured state.
3. **Tool-output hygiene.** Large tool results (a 2 MB JSON, a 50-page doc) never go into history verbatim. Store them externally, and put a short summary plus a handle (`result_id=af31`) in context. The agent can re-fetch when needed.
4. **Long-term store limits.** Cap items per user (e.g. 500 facts), dedupe on write, merge similar memories, apply TTL, and evict by `importance × recency`.
5. **Hierarchical summaries** for very long sessions: turn → segment summary → session summary.

```
 turns 1-40   ──► segment summaries (4 × 200 tokens)
 turns 41-60  ──► running summary (400 tokens)
 turns 61-66  ──► verbatim
 pinned facts ──► structured state (goal, IDs, constraints)
```

Example: in a 3-hour pairing session with an agent, context was growing past 150k tokens with P95 latency over 20 s. After moving tool outputs to handles and compacting at 60% of the window, working context stayed around 25k tokens, cost per turn dropped about 70%, and the goal-retention eval stayed at 97%.

**Follow-ups:**
- *When do you trigger compaction?* → At a threshold (e.g. 60–70% of budget) rather than at 100%, so there's room for the answer and a failure doesn't hit mid-turn.
- *Risk of compaction?* → Summary drift and lost details. Mitigate with pinned facts and by keeping the raw log retrievable.

**Red flags:**
- "Just use a model with a bigger context window."
- Truncating from the front with no summary.

---

## Q5. How do you handle long conversation history and context-window limits?
*Source: interview posts*

**What the interviewer is testing:** A layered context-assembly strategy with explicit budgets.

**Strong answer:**

I build context deliberately each turn instead of appending. The assembly order is:

```
[System prompt + policies]           fixed, cacheable prefix
[Pinned state: goal, user profile]   structured, small
[Running summary of older turns]     ~500-1000 tokens
[Relevant long-term memories]        top-k by similarity + recency
[Last N turns verbatim]              ~2-4k tokens
[Retrieved documents for this turn]  ~3-6k tokens
[Current user message]
[Reserved for output]
```

Techniques:
- **Token counting before the call**, using the model's tokenizer, with a budget per section. If over budget, reduce in priority order: fewer retrieved docs, then older verbatim turns, and never the system prompt or the pinned state.
- **Summarization / compaction** for older turns (see Q6–Q8).
- **Retrieval over history.** For very long histories, embed past turns and pull only relevant ones.
- **Put the stable prefix first** so prompt caching works. System prompt and tool definitions stay byte-identical across turns.
- **Place critical instructions at the start and repeat key constraints near the end**, because models attend less to the middle of long contexts.

Even with 200k–1M windows, I don't fill them by default. Cost scales per input token, TTFT grows with prefill length, and recall degrades in the middle. Big windows are a safety margin, not a strategy.

Example: a customer-support bot with average 25-turn sessions. The full-buffer approach cost about $0.09 per session at turn 25. Summary + window + entity memory brought it to about $0.03 with no drop in resolution rate.

**Follow-ups:**
- *What happens if you exceed the window anyway?* → The API errors or truncates. Always count tokens, and have a fallback compaction path.
- *How do you test it?* → Long-conversation eval: plant facts early, then query them late.

**Red flags:**
- Not knowing that input tokens are billed every turn.
- "1M context solves memory."

---

## Q6. When do you summarize past context instead of storing it in full?
*Source: interview posts*

**What the interviewer is testing:** The trade-off between fidelity and cost, and when lossy compression is acceptable.

**Strong answer:**

I keep things **verbatim** when exact wording matters:
- Recent turns (the model needs the exact phrasing to respond naturally).
- Legal, financial or medical statements, and anything that might be quoted back.
- Code, IDs, numbers, dates, addresses. Summaries mangle these.
- Anything needed for audit. That goes in the raw log, not necessarily in context.

I **summarize** when:
- The conversation exceeds my history budget (e.g. more than 20 turns or 8k tokens).
- Older turns carry *gist*, not detail: "we explored three options and rejected A and B because of cost".
- Tool traces and exploration steps whose outcome matters more than the path.
- Cross-session episodic memory, where the next session needs "what happened", not the transcript.

The rule of thumb: **summarize the narrative, extract the facts, keep the raw log.** A summary is for the model's context. Structured facts (IDs, amounts, decisions) are extracted into state so the summary can't lose them. The full transcript stays in cold storage for audit, debugging and re-summarization.

Example: in a loan-assistance bot, the rolling summary says "customer comparing a 20-year vs 15-year tenure, concerned about EMI". The structured state holds `loan_amount=₹42,00,000`, `rate=8.6%` and `preferred_emi<₹40,000`. The numbers never go through the summarizer.

**Follow-ups:**
- *How do you evaluate summary quality?* → QA-based checks: generate questions from the original, answer them from the summary, and measure coverage. Also check that critical entities are retained.
- *Abstractive or extractive?* → Abstractive for the narrative, extractive (copy key sentences) for compliance-sensitive content.

**Red flags:**
- Summarizing numbers and IDs through an LLM without extraction.
- Discarding the raw log.

---

## Q7. "Summarizing history with an LLM costs money too. Isn't that counterproductive?"
*Source: interview posts*

**What the interviewer is testing:** Whether you can do the cost maths instead of hand-waving.

**Strong answer:**

It's a fair challenge, so I'd answer with numbers. Without summarization, every turn resends the whole history. If each turn adds about 300 tokens, turn *n* sends about 300·n input tokens, so a 50-turn conversation sends roughly 300 × (1+2+…+50) ≈ **382k input tokens** in total.

With summarization every 10 turns: each call sends at most summary (500) + last 10 turns (3k) ≈ 3.5k tokens, so 50 turns ≈ **175k tokens**. On top of that there are 5 summarization calls of about 3.5k input and 500 output each, roughly 20k tokens. That's about **195k total, around half**. And the gap widens as conversations get longer, because full history is quadratic and summarization is roughly linear.

It also gets cheaper because:
- **The summarizer can be a small model** (a Haiku/Flash-class model) at a fraction of the main model's price per token, so the summarization calls' cost is small.
- **Run it asynchronously**, after the response is sent, so it adds zero user-facing latency.
- **Summarize incrementally.** Update the previous summary with only the new turns rather than re-summarizing everything.
- **Trigger it only when needed.** Sessions under the threshold (most of them, e.g. 80% of support chats end in under 10 turns) never pay the cost.
- **Prompt caching** on the stable prefix reduces the per-turn cost of the non-summary path too.

Plus, it isn't only about cost: shorter context means lower TTFT and fewer "lost in the middle" errors.

When it *is* counterproductive: short sessions, or when you need exact recall of everything. Then use a sliding window plus extraction, or retrieval over the full log.

**Follow-ups:**
- *Doesn't summarization lose information?* → Yes, which is why facts go to structured state and the raw log stays retrievable.
- *Where would you run it?* → A background worker after the response, triggered by a token threshold.

**Red flags:**
- "Yes, it's more expensive, but it's necessary", with no numbers.
- Using the same large model for summarization, synchronously.

---

## Q8. How would you architect real-time conversation summarization?
*Source: interview posts*

**What the interviewer is testing:** A system design with async processing, consistency and failure handling.

**Strong answer:**

Requirements first: summaries must be fresh enough for the next turn, add no user-facing latency, never lose facts, and survive failures.

```
 user msg ──► Chat API ──► build context (summary + last N turns + facts)
                 │                    ▲
                 │ response streamed  │ read
                 ▼                    │
          append turn to log ──► Redis/DB (session store)
                 │
                 └─► event: turn_appended(session_id, token_count)
                                 │
                        Summarizer worker (queue)
                        - if unsummarized tokens > threshold:
                          new_summary = small_llm(prev_summary + new turns)
                          extract facts → structured state
                          write with version check (optimistic lock)
```

Design points:
- **Incremental, not full**: `summary_v(n+1) = f(summary_v(n), turns[k..m])`. Store `summarized_up_to_turn` so the context builder knows which turns to include verbatim.
- **Async and off the hot path.** Triggered via a queue. If the worker is behind, the context builder simply includes more verbatim turns, so there's graceful degradation and no blocking.
- **Consistency:** optimistic locking on the summary version, so two workers can't clobber each other. Summarization is idempotent per `(session, up_to_turn)`.
- **Fact extraction alongside the summary.** The same call returns JSON with `summary` plus `facts[]`, validated with Pydantic.
- **For voice or live calls:** summarize every N seconds or on speaker turns, keep a very small model, and stream updates to an agent-assist UI.
- **Quality guardrails:** entity-retention check (all IDs and amounts in the new turns must appear in the facts or the summary). On failure, keep the old summary and more verbatim turns.
- **Observability:** summary lag (turns behind), summarization cost per session, and an eval on sampled sessions.

Example numbers: a contact center with 5k concurrent calls. A Flash-class summarizer every ~60 s per call is about 80 calls/s, at under 1% of total LLM spend, with a P95 summary lag under 5 s.

**Follow-ups:**
- *What if summarization fails?* → Fall back to a longer verbatim window, retry with backoff, and alert if the lag grows.
- *How do you handle a user correction ("actually, it's 3 units, not 2")?* → Fact extraction updates the keyed fact, and the next summary incorporates the correction.

**Red flags:**
- Synchronous summarization before every response.
- Re-summarizing the entire transcript each time.

---

## Q9. How do you maintain context in a long-running chatbot?
*Source: interview posts*

**What the interviewer is testing:** An end-to-end approach combining session state, memory and retrieval across sessions and devices.

**Strong answer:**

"Context" covers three things. **Within a turn**, the model needs the right context. **Within a session**, the conversation should feel continuous. **Across sessions**, the bot should remember the user. I handle each:

1. **Session store** (Redis or Postgres) keyed by `session_id`: messages, running summary, structured state (current intent, slots filled, open tasks). It's persisted, so a reconnect or a server restart doesn't lose the session. Chat servers stay stateless.
2. **Structured slots and state.** For a task-oriented bot, I track a state machine or slots (`intent=refund`, `order_id=…`, `step=awaiting_confirmation`). This is more reliable than asking the LLM to re-infer the state from history each turn.
3. **Query condensation.** Every follow-up is rewritten into a standalone query using recent turns before retrieval.
4. **Rolling summary + window** for older turns (Q5–Q8).
5. **Long-term user memory** loaded at session start: profile facts plus the top-k relevant episodes.
6. **Topic-shift handling.** Detect when the user switches topic and start a new "thread" segment, so irrelevant old context doesn't pollute retrieval.
7. **Resumption.** When a user returns after 2 days, greet with a short recap from the episodic summary: "Last time we were setting up your GST invoice template, shall we continue?"

Example: a banking assistant used across web and mobile. The session state lived in Redis with a 30-min TTL and was flushed to Postgres on expiry. Long-term facts sat in a user profile table, and episodes were in pgvector. The multi-turn task completion rate went from 71% to 86% once slots moved from "LLM infers from history" to explicit state.

**Follow-ups:**
- *Stateless servers but stateful conversations, how?* → Externalize all state keyed by session. Any instance can serve any turn.
- *How do you test it?* → Scripted multi-turn evals with follow-ups, topic switches and corrections.

**Red flags:**
- Storing session state in server memory.
- No query rewriting, so follow-up questions retrieve garbage.

---

## Q10. How do you handle conflicting information in memory?
*Source: interview posts*

**What the interviewer is testing:** Conflict resolution strategy: recency, source trust, provenance, and asking the user.

**Strong answer:**

Conflicts are inevitable. The user moves city, changes preference, or a document gets updated. I deal with them at **write time** and at **read time**.

**At write time (preferred):**
- Store facts as **keyed records**, `(entity, attribute) → value`, with `source`, `timestamp`, `confidence` and `version`. A new value for the same key **supersedes** the old one. The old one is kept in history, not deleted, for audit.
- Before writing a free-text memory, retrieve similar memories and ask a small model: "Does the new memory duplicate, update, or contradict any of these?" Then do ADD / UPDATE / DELETE / NOOP accordingly. This is the pattern tools like Mem0 use.

**At read time:**
- If conflicting items still get retrieved, resolve them with rules: **explicit user statement > system-of-record data > inferred**, then **newer > older**.
- Include the timestamp in the context ("as of 2026-03-02: user located in Pune") so the model can reason about recency.
- If confidence is low and the fact matters (a shipping address, an allergy), **ask the user** rather than guess.

**Source of truth matters.** Memory should never override the system of record. If the CRM says the address is X and memory says Y, the CRM wins, and memory should store a pointer rather than a copy.

Example: a travel agent had "vegetarian" from 2024 and "eats fish now" from 2026. With keyed facts and supersession, only the latest preference is injected. Previously both were retrieved, and the model hedged in about 15% of meal-related responses.

**Follow-ups:**
- *What about conflicts between memory and retrieved documents?* → Documents are authoritative for org knowledge. Memory is authoritative for user preference.
- *How do you detect contradictions at scale?* → Periodic batch job: cluster memories per user, and run an NLI or LLM contradiction check.

**Red flags:**
- "The LLM will figure out which one is right."
- Hard-deleting old values with no audit trail.

---

## Q11. What are the common challenges of memory in long-running agentic systems?
*Source: interview posts*

**What the interviewer is testing:** Awareness of real failure modes from production, not theory.

**Strong answer:**

The challenges I've seen, and the mitigation for each:

| Challenge | What happens | Mitigation |
|---|---|---|
| **Context bloat** | Tool outputs pile up; cost and latency explode | Handles for large outputs, compaction thresholds |
| **Goal drift** | After 40 steps the agent forgets the original objective | Pinned goal and constraints in state, re-injected every step |
| **Summary drift / error compounding** | Each summary of a summary loses detail or introduces errors | Incremental summaries, fact extraction, raw log retained |
| **Stale memory** | Acts on facts that were true last month | TTLs, timestamps, re-verify with tools before acting |
| **Memory poisoning** | Injected content from a web page gets saved as a "fact" and influences future sessions | Never auto-save from untrusted tool content; provenance tags; review |
| **Cross-user leakage** | Retrieval without a tenant filter returns another user's memory | Mandatory `user_id`/`tenant_id` filter enforced at the data layer |
| **Crash recovery** | Worker dies at step 37 and state is lost | Checkpoint after every step (e.g. LangGraph checkpointer on Postgres) |
| **Concurrency** | Parallel sub-agents write conflicting state | Reducers or merge functions, per-key ownership, optimistic locks |
| **Unmeasurable quality** | Nobody knows whether memory is helping | Memory-specific evals: retention, correct-recall, harmful-recall |
| **Privacy and retention** | Memory holds PII indefinitely | Redaction, TTL, delete-by-user support |

The meta-point: memory makes the system **stateful**, and stateful systems need the usual engineering disciplines: schemas, migrations, access control, backups, observability.

Example: a research agent on multi-hour tasks started re-running searches it had already done after about 2 hours. The cause was that compaction had dropped the "already searched" list. We moved it into structured state (`visited_queries: set`), which cut duplicate tool calls by 40%.

**Follow-ups:**
- *Which is most dangerous?* → Memory poisoning and cross-user leakage, because they're security incidents, not quality bugs.
- *How do you checkpoint efficiently?* → Store state deltas per step, and keep large blobs in object storage by reference.

**Red flags:**
- Only mentioning "context window limit".
- No awareness of security implications of persistent memory.

---

## Q12. A user says "forget everything about me". How does your memory design support that (privacy/GDPR)?
*[Added]*

**What the interviewer is testing:** Privacy-by-design, and whether deletion is actually possible across every store.

**Strong answer:**

Deletion is hard if you didn't design for it, because user data ends up spread across session logs, summaries, vector embeddings, caches, traces and possibly fine-tuning datasets. So the design principle is: **every memory artifact is keyed by `user_id` (and `tenant_id`) from day one.**

Supporting "forget me":
1. **Inventory the stores:** session store, long-term fact table, vector DB (episodic memory), semantic cache, observability traces (LangSmith/Langfuse), analytics warehouse, eval datasets, backups.
2. **Delete by key** in each: `DELETE WHERE user_id=…` in SQL and a filter-based delete in the vector DB (Qdrant, Pinecone and pgvector all support delete-by-metadata). Invalidate cache entries tagged with the user.
3. **Traces:** redact PII at ingestion so traces hold little personal data, and set retention (e.g. 30 days).
4. **Backups:** document the retention window. Deleted data ages out of backups within N days, which is the accepted GDPR practice.
5. **Fine-tuning data:** never train on raw user memory without consent. If you did, you'd have to retrain, which is why consent and anonymization happen *before* training.
6. **Verification:** a deletion job that emits an audit record, plus a test that confirms the user's data is no longer retrievable.
7. **Granular controls:** let users view and delete individual memories ("forget that I live in Pune"), not just all-or-nothing. Many assistants expose a "memory" settings page.

Also the flip side: **consent and transparency** at write time. Tell users what is remembered, and offer a memory-off mode.

Example: we implemented a `forget_user` workflow that fans out to 6 stores. It ran P95 under 2 minutes and wrote an audit log entry that legal could show to the DPO.

**Follow-ups:**
- *Can you "delete" from embeddings?* → Yes, delete the vectors themselves. Embeddings can leak content, so treat them as personal data.
- *What about data processed by a third-party LLM provider?* → Use zero-data-retention agreements, and route by region for residency.

**Red flags:**
- "We just delete the chat history" (ignoring embeddings, caches and traces).
- Treating embeddings as anonymous.

---

## Rapid-fire recap

- LLMs are stateless; memory is application-managed context.
- Working state is per task; semantic memory holds facts; episodic memory holds past experiences; procedural memory holds how to do things.
- Default pattern: pinned state + rolling summary + last N turns + retrieved memories + retrieved docs.
- Summarize the narrative, extract the facts, keep the raw log.
- Full history is quadratic in cost; summarization with a small async model is roughly linear.
- Large tool outputs go into context as handles, not contents.
- Store facts as keyed records so updates supersede old values; resolve conflicts by source trust, then recency.
- Never auto-save untrusted content, because memory poisoning persists across sessions.
- Enforce user and tenant filters at the data layer for every memory read.
- Checkpoint agent state every step so crashes resume instead of restarting.
- Design deletion from day one: embeddings, caches and traces are personal data too.
- Evaluate memory with plant-and-recall tests over long conversations.
