# 22 — Strategy, Leadership & Project Deep-Dive

**What interviewers probe here:** ownership and judgment. Can you explain your own system clearly and defend its trade-offs? Do you understand organizational and strategic choices, and can you talk honestly about failures? For senior roles, this round often decides the level you're hired at.

[← Back to index](README.md)

1. [Explain your GenAI project architecture](#q1)
2. [POC to production challenges](#q2)
3. [Build vs buy](#q3)
4. [Team structure: model, data, platform](#q4)
5. [Communicating AI risk to leadership](#q5)
6. [Defending a decision](#q6)
7. [Questions to ask the interviewer](#q7)
8. [An AI failure you owned](#q8)
9. [Stakeholder demands 100% accuracy](#q9)
10. [Keeping up without chasing hype](#q10)

> Structure for any project answer: **Problem → Users & constraints → Architecture → Key decisions & trade-offs → How you measured it → Results → What you'd change.** Use real numbers; interviewers remember them.

---

<a id="q1"></a>
## Q1. Explain your GenAI project architecture end to end.
*Source: interview posts*

**What the interviewer is testing:** Clarity, depth on your own decisions, and whether you actually built it.

**Strong answer (worked example to adapt):**

"**Problem:** our support team of 120 agents handled about 40k tickets a month, and 55% were policy or how-to questions answered from a 6,000-page knowledge base. Median first response was 4 hours.

**Constraints:** answers had to be grounded, with citations; the knowledge base changed daily; customer data stayed in India; the budget was ₹8 lakh per month; P95 latency under 5 s.

**Architecture:**
- **Ingestion:** Confluence and PDF connectors run on change events. We did layout-aware parsing (tables kept as markdown), heading-aware chunks of about 400 tokens with 15% overlap, metadata (product, version, region, ACL groups), and embeddings with a multilingual model, stored in Qdrant.
- **Query path:** a FastAPI service. First query rewriting using the conversation. Then hybrid retrieval (BM25 + dense, fused with RRF), metadata filters (product and ACL), and a cross-encoder rerank of the top 30 down to 6. Generation used a mid-tier model with a strict grounding prompt and required citations, then a groundedness check with a small NLI model before the response streamed over SSE.
- **Agentic part:** only two tools, order lookup and ticket creation. Orchestrated in LangGraph, and ticket creation needed agent confirmation.
- **Ops:** Langfuse tracing, a semantic cache scoped per product, a nightly eval suite of 350 questions with RAGAS plus custom checks, and dashboards for cost, latency and deflection.

**Key trade-offs:** we chose RAG over fine-tuning because facts changed daily. We added a reranker, which cost 120 ms but raised Recall@5 from 71% to 86%. We used a mid-tier model instead of a frontier one, since evals showed only a 2-point quality difference at a third of the cost.

**Results:** 38% of tickets were deflected with a CSAT of 4.3/5. First response for the rest dropped to 1.5 hours because agents got drafted answers. Cost was about ₹5.5 lakh per month.

**What I'd change:** move evals to include production samples earlier. For the first month our offline set didn't contain the Hinglish queries that made up 20% of traffic."

**Follow-ups:**
- *Why Qdrant?* → Payload filtering for ACLs, self-hostable in-region, and our scale (about 2M chunks) fits one cluster.
- *Hardest bug?* → Stale chunks after page edits; we fixed it with versioned document IDs and delete-then-upsert.

**Red flags:**
- Listing tools without saying why each was chosen.
- "We" for everything, with no clarity on your own contribution.
- No numbers.

---

<a id="q2"></a>
## Q2. What challenges did you face moving from POC to production?
*Source: interview posts*

**What the interviewer is testing:** Real production experience: the unglamorous 80% of the work.

**Strong answer (STAR):**

- **Situation:** "Our HR policy bot demo worked well on 20 hand-picked questions, and leadership wanted it rolled out to 8,000 employees in 6 weeks."
- **Task:** "I owned taking it to production: quality, security and operations."
- **Action:** the challenges and what I did about each:
  1. **The eval gap:** the demo set didn't match real traffic. I sampled 400 real questions from the pilot and labelled them with HR. Accuracy was 64%, not the "90%" we'd assumed.
  2. **Messy documents:** scanned PDFs, tables and outdated policy versions. I added OCR, table-aware parsing and an effective-date filter, and deduped versions. That took accuracy to 79%.
  3. **Access control:** manager-only policies had been visible to everyone. I added ACL metadata from AD groups and pre-filtered at retrieval.
  4. **Latency and cost:** P95 was 11 s. Parallel retrieval, a reranker on a smaller candidate set, streaming and prompt caching brought it to 3.8 s at 40% lower cost.
  5. **Refusals:** the bot guessed on policies it didn't have. I added a grounded-refusal path plus handoff to HR, and a groundedness check.
  6. **Operations:** tracing, alerts, a feedback button, an owner rota, and an incident runbook.
- **Result:** "We launched in 7 weeks at 87% accuracy on the real eval set, with 62% of HR queries resolved without a ticket. There were no access-control incidents in the first 6 months."
- **Learning:** "A POC proves possibility; production proves reliability. I now build the eval set from real traffic in week one."

**Follow-ups:**
- *What would you cut if you had 3 weeks?* → Launch to one department, read-only, with human review of low-confidence answers.

**Red flags:**
- "It was mostly straightforward."
- Only talking about model or prompt tweaks.

---

<a id="q3"></a>
## Q3. How do you decide build vs buy for a core AI capability?
*Source: interview posts*

**What the interviewer is testing:** Strategic reasoning: differentiation, TCO, risk and time.

**Strong answer:**

I'd score the options on these criteria:

| Criterion | Lean build | Lean buy |
|---|---|---|
| Differentiation | Core to the product's value, a competitive moat | Commodity capability (OCR, transcription, generic chat) |
| Data | Proprietary data and feedback loops are the advantage | Generic data is enough |
| Fit | Unique workflows, integrations, domain constraints | Standard workflow |
| Control & compliance | Strict residency, auditability, on-prem needs | Vendor meets certifications |
| Team | Have or can hire ML and platform talent | No team, or it's needed elsewhere |
| Time to value | Can wait 3–6 months | Need it this quarter |
| TCO (3-year) | Lower at our scale | Lower at our scale |
| Vendor risk | Lock-in, pricing changes, vendor viability are concerns | Stable vendor, exit path exists |

The usual answer is **hybrid**: buy the commodity layers (foundation models via API, a vector DB, OCR, an observability tool) and build the differentiating layer (domain retrieval, workflow orchestration, evals, and the data flywheel). Keep an abstraction layer (a model gateway, your own interfaces) so you can swap vendors.

TCO honesty: "build" costs include maintenance, on-call, security reviews and keeping up with the field. "Buy" costs include per-seat or per-call fees at scale, integration effort, and lost learning.

Example: for payroll-query automation at an HR-tech company, we bought the LLM (API) and OCR, and built the payroll-rule retrieval plus the eval harness. Payroll logic was our differentiation, and no vendor understood Indian statutory compliance (PF, ESI, PT variations by state) well enough.

**Follow-ups:**
- *When would you revisit the decision?* → When volume makes API cost exceed self-hosting TCO, or when the vendor's roadmap diverges from your needs.

**Red flags:**
- "Always build, it's cheaper" or "always buy, it's faster", with no criteria.

---

<a id="q4"></a>
## Q4. How would you structure a team around model, data and platform ownership?
*Source: interview posts*

**What the interviewer is testing:** Org design that avoids ownership gaps and bottlenecks.

**Strong answer:**

It depends on scale, but a pattern that works:

- **AI Platform team (horizontal):** the LLM gateway (routing, keys, rate limits, cost attribution), the eval framework, observability and tracing, vector/retrieval infrastructure, guardrails as a service, prompt and model registry. It is measured by the adoption, reliability and cost efficiency of what product teams build.
- **Data team (horizontal or embedded):** ingestion pipelines, document quality, metadata and ACL sync, labelling and golden datasets, data governance. It is measured by freshness, coverage and data quality SLAs.
- **Applied AI / product squads (vertical):** each owns a user-facing AI feature end to end (prompts, retrieval tuning, agent logic, feature-level evals, product metrics). Each squad has an AI engineer, backend and frontend engineers, a PM and a designer. They are measured by business outcomes.
- **Model team (only if justified):** fine-tuning, distillation, self-hosted serving. Only when you have a real need, such as scale economics, latency or on-prem requirements.
- **Governance:** Responsible AI, security and legal as a review function with clear SLAs, not a blocker at the end.

Principles:
- **Clear ownership of quality:** the product squad owns feature quality; the platform provides the tools. Otherwise quality becomes "someone else's problem".
- **Paved road:** the platform makes the right thing easy (default tracing, evals and guardrails), so squads don't reinvent it.
- **Start embedded, then extract:** with 1–2 AI features, keep people in the squads; around 4+ features, extract a platform team when duplication appears.
- An **AI guild** for shared learning and standards.

**Follow-ups:**
- *Who owns an incident where the bot gave wrong answers?* → The squad leads; platform and data support based on the root cause, with a shared postmortem.

**Red flags:**
- A central "AI team" that builds everything for everyone, which becomes a bottleneck.

---

<a id="q5"></a>
## Q5. How do you communicate AI risk and limitations to non-technical leadership?
*Source: interview posts*

**What the interviewer is testing:** Translating probabilistic behavior into business terms without fear-mongering or overselling.

**Strong answer:**

1. **Lead with the business decision, not the technology.** "We can automate 60% of tier-1 queries. The question is what error level we accept and where humans stay involved."
2. **Use concrete frequencies, not percentages alone.** "About 1 in 50 answers will contain an error. At 10k queries per day that's 200 wrong answers daily. Most are minor, and about 10 could be misleading on billing."
3. **Compare with the current baseline.** "Human agents currently make errors on about 4% of tickets; the AI is at 2% on the eval set." Risk is relative.
4. **Categorize by severity:** an inconvenience (a slightly off answer), a financial risk (a wrong refund promise), a compliance risk (data exposure). Give the mitigation for each.
5. **Show the controls:** what's automated vs human-reviewed, monitoring, a kill switch, rollback time ("we can turn it off in 5 minutes").
6. **Be explicit about unknowns:** "It hasn't been tested on Tamil queries yet; we're launching in English first."
7. **Give them the decision:** options with trade-offs (A: faster launch, more human review; B: 4 weeks later, broader automation).
8. **Report regularly** with a simple dashboard of quality, incidents, cost and business value.

Example: "Instead of saying 'LLMs hallucinate', I showed the CXO team 5 real failure examples from our eval, what each would cost if it reached a customer, and the guardrail that catches it. They approved launch with human review on billing topics. That was a better decision than either blocking the launch or allowing full automation."

**Follow-ups:**
- *What if leadership wants to skip the controls?* → Document the risk acceptance with a named owner, and propose a staged rollout.

**Red flags:**
- Jargon ("the embedding recall is low").
- Overselling ("it's basically 100% accurate").

---

<a id="q6"></a>
## Q6. Defend a decision: why did you choose it, what alternatives did you consider, how did you prove it works, what changes at scale, what trade-offs did you accept?
*Source: interview posts*

**What the interviewer is testing:** Engineering judgment. This is the core skill the posts say interviews now test.

**Strong answer (use this five-part template on any decision):**

Worked example: **"Why hybrid search plus a reranker instead of pure vector search?"**

1. **Why:** "Our queries mixed natural language ('how do I reset my password') with exact identifiers (error codes like `E-4012`, SKU numbers). Pure dense retrieval missed exact identifiers. On our eval set, Recall@5 was 58% for ID-bearing queries."
2. **Alternatives considered:**
   - Pure BM25: good for IDs, poor on paraphrases (Recall@5 of 62% overall).
   - Fine-tuning the embedding model: promising, but it needed labelled pairs we didn't have and a re-index cycle.
   - A larger embedding model: +3 points, and double the storage.
   - Hybrid with RRF plus a cross-encoder reranker: chosen.
3. **How I proved it:** "An offline eval on 500 labelled queries, split by query type. Hybrid plus the reranker reached Recall@5 of 88%, compared with 74% for dense only. End-to-end answer correctness went from 76% to 85%. Then an online A/B on 10% of traffic for 2 weeks: thumbs-down rate fell from 9% to 6%."
4. **What changes at scale:** "The reranker is the latency and cost hotspot, at about 120 ms for 30 candidates on a GPU. At 10x traffic I'd reduce the candidates to 20, batch reranker calls, consider a distilled reranker, and cache frequent queries. The BM25 index needs sharding past about 50M chunks."
5. **Trade-offs accepted:** "120 ms more latency, one more service to operate, and tuning complexity (RRF weights). We accepted them because retrieval quality was the main driver of answer errors."

Practise this template on 3–4 major decisions in your project: model choice, chunking, framework, and whether to use an agent at all.

**Follow-ups:**
- *What would make you reverse the decision?* → If a new embedding model closed the ID-recall gap on its own, I'd drop BM25 to simplify.

**Red flags:**
- "It's best practice."
- No alternatives, no measurement.

---

<a id="q7"></a>
## Q7. What should you ask the interviewer?
*Source: interview posts*

**What the interviewer is testing:** Curiosity, seniority, and whether you evaluate them too.

**Strong answer:**

Ask questions that show how they actually build, not just about perks. Pick 2–4:

**The reality check** (from the posts):
- "Take one of the questions you asked me, say how you catch silent regressions or handle stale documents. How does your team handle it in your own system today?" The honesty and specificity of the answer tells you what the org actually values versus what it says it values. "We don't yet, and it's a priority" is a good answer; hand-waving is a signal.

**Engineering maturity**
- "How do you evaluate AI features before release? Is there an eval suite in CI?"
- "What does an AI incident look like here, and how was the last one handled?"
- "How do you decide between prompting, RAG, fine-tuning, or not using AI at all?"
- "What's the split between building new features and maintaining existing ones?"

**Ownership and impact**
- "What would success look like for this role in 6 months?"
- "Who owns production quality of AI features: the product squad or a central team?"
- "What's the biggest unsolved technical problem the team has?"

**Business and data**
- "How is AI value measured here: usage, cost savings, revenue?"
- "What data access will I have, and how long does it take to get it?"

**Team**
- "How do you keep up with model changes? Who decides to upgrade models?"

Avoid questions answered on the careers page, and don't open with salary in a technical round.

**Follow-ups:**
- *What if they give a vague answer?* → Ask gently for a concrete example: "Could you walk me through the last time that happened?"

**Red flags:**
- "No questions."
- Only asking about remote work and leave policy in a technical round.

---

<a id="q8"></a>
## Q8. Tell me about an AI system failure you owned.
*[Added]*

**What the interviewer is testing:** Accountability, debugging depth and systemic learning.

**Strong answer (STAR, worked example):**

- **Situation:** "Two weeks after launch, our sales-assistant voice agent told a real-estate prospect a flat price, a unit configuration and a bank tie-up. None of it existed in the client's knowledge base. The client escalated."
- **Task:** "I was the owning engineer. I needed to find the root cause, stop recurrence, and restore the client's trust."
- **Action:**
  - "I pulled the trace. Retrieval had actually returned relevant chunks at 0.85+ similarity, so the model wasn't simply ignoring context."
  - "I compared the final prompt sent to the model with the one I expected. Two bugs: (1) the anti-fabrication instruction was defined but never attached on the live code path, and (2) retrieval ran fire-and-forget, so the prompt was assembled **before** the results arrived. The model answered with zero context and filled the gaps from its priors."
  - "Fixes: await retrieval with a timeout budget; if no context arrives, a deterministic 'I don't have that detail; let me connect you to our sales team' path; a test asserting that the rendered prompt contains the grounding rule and a non-empty context; and a groundedness check on numeric claims."
  - "I added a daily sample-and-judge job for fabricated entities, and alerting."
- **Result:** "Fabricated-fact incidents went to zero over the next 3 months across 40k calls. The client stayed, and the 'prompt contract' tests became a team standard."
- **Learning:** "Most 'hallucinations' in production are plumbing bugs. I now log and inspect the **exact final prompt**, not the intended one."

**Follow-ups:**
- *What did you do for the client?* → A same-day incident summary, a call-back to the affected prospect with correct information, and a written RCA.

**Red flags:**
- Blaming the model or another team.
- A trivial failure with no lesson.

---

<a id="q9"></a>
## Q9. A stakeholder demands 100% accuracy. How do you respond?
*[Added]*

**What the interviewer is testing:** Reframing expectations constructively.

**Strong answer:**

I wouldn't say "that's impossible" and stop there. I'd reframe the request around risk:

1. **Understand the underlying need.** Usually it's "I can't afford wrong answers on X." Find X: prices, legal claims, medical dosages.
2. **Compare with the baseline.** "Today's process has about a 3% error rate. Our target is lower than that, with errors caught before they reach customers."
3. **Engineer for correctness where it matters:**
   - Deterministic paths for critical facts: prices come from the database via a tool, never generated.
   - Abstention: the system says "I don't know" and hands off when confidence or grounding is low, trading coverage for precision. "It answers 70% of questions at 99% precision, and routes 30% to humans."
   - Human review for high-stakes categories.
4. **Define measurable targets:** precision on critical categories, abstention rate, time to correct.
5. **Agree on error handling:** monitoring, feedback and fast correction.

Example: "For a pharmacy chatbot, dosage questions were never generated; they returned the exact label text from a verified database or went to a pharmacist. The 'always correct' requirement was met for the critical subset, and general questions used RAG with citations."

**Follow-ups:**
- *What if they insist?* → Then that use case shouldn't be automated with an LLM. Say so clearly and offer the assistive version (drafts for humans).

**Red flags:**
- Promising 100%.
- Dismissing the concern.

---

<a id="q10"></a>
## Q10. How do you keep up with the field without chasing hype?
*[Added]*

**What the interviewer is testing:** Learning discipline and judgment about new tech.

**Strong answer:**

- **Filter by problem:** I track developments relevant to problems we actually have (retrieval quality, agent reliability, cost). A new framework matters only if it solves one of them.
- **Sources:** model provider release notes and docs, a few strong engineering blogs and papers (I read abstracts weekly and deep-read one or two), eval leaderboards with scepticism, and practitioner communities.
- **Evaluate on our data:** every "this model is better" claim runs through our eval suite before it gets opinions. Example: a new model announced as "SOTA on reasoning" scored 1 point better on our eval, at 2x the latency, so we didn't switch. A cheaper mini model matched our quality on classification tasks, so we moved routing to it and saved 60%.
- **Time-boxed spikes:** 1–2 days to prototype a promising technique, with a written verdict (adopt / watch / skip).
- **Fundamentals over tools:** attention, retrieval, evaluation and distributed-systems basics don't expire; frameworks do.
- **Share:** internal write-ups and demos, since teaching forces understanding.

**Follow-ups:**
- *Something you adopted recently?* → Give a real example with a measured result.

**Red flags:**
- "I follow AI influencers."
- Listing ten tools with no evaluation criteria.

---

## Rapid-fire recap

- Project answer: problem → constraints → architecture → decisions → measurement → results → what you'd change.
- Real numbers (latency, accuracy, cost, volume) make answers credible.
- POC to production is mostly evals on real traffic, data quality, access control, latency and ops.
- Build what differentiates, buy commodities, and keep an abstraction layer for exit.
- Product squads own feature quality; the platform provides a paved road; the data team owns freshness and quality.
- For leadership: frequencies, a baseline comparison, severity tiers, controls, and a decision to make.
- Defend decisions with why, alternatives, proof, scale and trade-offs.
- Ask interviewers how they handle these problems themselves; honesty reveals culture.
- Failure stories: own it, show the debugging, show the systemic fix.
- "100% accuracy" becomes deterministic paths plus abstention plus human review for critical facts.
- Test new models and tools on your own evals before forming opinions.
