# AI Engineer Interview Prep — Scenario-Based Question Bank

This bank collects questions from about 30 LinkedIn posts: real interview experiences, "most repeated questions" lists, cheat sheets and roadmaps. **Duplicates and near-duplicates are merged**, so each question below appears once, in its best-fit topic. Questions marked **[Added]** are extra ones that real interviews commonly ask but the posts didn't cover.

## How each answer is written

Every question follows the same format:

| Section | Purpose |
|---|---|
| **What the interviewer is testing** | The skill the question is really probing |
| **Strong answer** | What a strong candidate would say out loud: clarify → approach → trade-offs → a concrete example with numbers |
| **Follow-ups** | Likely next questions, with short answers |
| **Red flags** | Things weak candidates say that lose the interviewer |

> Rule of thumb from the posts: most of these questions have **no single right answer**. Interviewers reward candidates who give options, pick one, and **defend the trade-off**.

## Topic files

| # | File | Focus |
|---|---|---|
| 01 | [LLM Fundamentals](01-llm-fundamentals.md) | Tokens, attention, prefill, decoding, determinism, repetition |
| 02 | [Prompt & Context Engineering](02-prompt-context-engineering.md) | Prompt strategies, context engineering, caching, structured output, constrained decoding |
| 03 | [Embeddings & Vector Databases](03-embeddings-vector-databases.md) | Embedding models, dimensions, ANN, vector DB selection, local vs prod |
| 04 | [RAG Core Pipeline](04-rag-core-pipeline.md) | Chunking, hybrid search, BM25, RRF, reranking, citations, tables/OCR |
| 05 | [Advanced RAG](05-advanced-rag.md) | CRAG, Self-RAG, GraphRAG, HyDE, RAG Fusion, contextual retrieval, multi-hop |
| 06 | [RAG in Production & Data Freshness](06-rag-production-data-freshness.md) | Re-indexing, versioning, permissions, multi-tenancy, 50M-doc ingestion, latency |
| 07 | [RAG Debugging & Hallucinations](07-rag-debugging-hallucinations.md) | Accuracy drops, right doc → wrong answer, hallucination root causes |
| 08 | [Evaluation](08-evaluation.md) | RAGAS, retrieval metrics, LLM-as-judge, eval at scale, statistics, agent eval |
| 09 | [Agents: Fundamentals & Tool Calling](09-agents-fundamentals-tool-calling.md) | Agent vs workflow, function calling internals, ReAct, planning, tool routing |
| 10 | [Agent Reliability in Production](10-agent-reliability-production.md) | Loops, retries, idempotency, checkpointing, HITL, cost limits, versioning |
| 11 | [Memory & Context Management](11-memory-context.md) | Short/long-term, episodic/semantic memory, summarization, unbounded growth |
| 12 | [Multi-Agent Systems](12-multi-agent-systems.md) | Single vs multi, supervisor vs peer, handoffs, race conditions, failure isolation |
| 13 | [Agent Frameworks & MCP](13-agent-frameworks-mcp.md) | LangGraph, CrewAI, ADK, Claude Agent SDK, custom orchestration, MCP |
| 14 | [Fine-Tuning, Model Selection & Optimization](14-fine-tuning-model-selection.md) | RAG vs FT, LoRA, distillation, quantization, open vs closed, model routing |
| 15 | [LLM Inference & Serving](15-llm-inference-serving.md) | KV cache, vLLM params, HPA, latency metrics, SSE, GPU sizing |
| 16 | [Production System Design](16-production-system-design.md) | Cost, caching, observability, gateway, multi-region, failover, provider migration |
| 17 | [Security, Safety & Guardrails](17-security-safety-guardrails.md) | Prompt injection, PII, authZ, tool permissions, destructive actions |
| 18 | [NL-to-SQL & Structured Data](18-nl-to-sql-structured-data.md) | Text-to-SQL correctness, semantic layers, safety |
| 19 | [Classical ML Fundamentals](19-classical-ml.md) | Imbalanced data, micro/macro F1, metrics |
| 20 | [Python, Backend & Coding](20-python-backend-coding.md) | Async, FastAPI, connection pools, Kadane, build-from-scratch exercises |
| 21 | [Domain Scenarios: AI in Supply Chain](21-domain-scenarios-supply-chain.md) | AI decision governance, autonomy limits, ROI |
| 22 | [Strategy, Leadership & Project Deep-Dive](22-strategy-leadership-behavioral.md) | Build vs buy, team structure, communicating risk, POC→prod, defending decisions |

---

## Master question index (deduplicated)

### 01 — LLM Fundamentals
1. What happens between sending a prompt and the LLM generating its first token?
2. Why is the attention mechanism important? Explain self-attention (Q/K/V, multi-head) simply.
3. What is the difference between tokenization and embeddings?
4. Prove why GPT can't reliably count the r's in "strawberry".
5. What do tokens, parameters and context windows mean in practice for cost, latency and limits?
6. Explain decoding strategies: greedy, temperature, top-k, top-p, beam search.
7. An LLM gives different answers to the same prompt. How do you make the output deterministic and consistent?
8. Why do LLMs sometimes repeat the same sequence, and how do you mitigate it?
9. Why do embeddings work at all?
10. AI vs ML vs DL vs GenAI vs LLM vs Agent vs Agentic AI: give crisp distinctions.
11. [Added] Encoder-only vs decoder-only vs encoder-decoder. Why are most LLMs decoder-only?
12. [Added] How do positional encodings (RoPE) work, and why does long context degrade?
13. [Added] Pretraining vs instruction tuning vs RLHF/DPO: what does each stage give the model?
14. [Added] Why do LLMs hallucinate at a fundamental level?
15. [Added] Reasoning models and chain-of-thought: when is test-time compute worth the cost?

### 02 — Prompt & Context Engineering
1. What prompt strategies do you know (zero-shot, few-shot, CoT, ReAct, role, structured), and when do you use each?
2. How do you make prompt design a reproducible engineering task instead of trial and error?
3. What is context engineering? How do you balance completeness against token efficiency and handle retrieval noise and context collapse?
4. How do you apply guardrails at the prompt level, and why isn't the prompt alone enough?
5. What is prompt caching, and how do you structure prompts to benefit from it?
6. How do you get reliable structured (JSON) output from an LLM?
7. Your output schema is fully known in advance. Why make the model generate it token by token? Explain constrained / parallel decoding with KV-cache reuse.
8. "LLM writes, decision model decides, code acts": when would you use a small classifier/decision model instead of an LLM for routing, classification or validation?
9. [Added] A prompt change fixed one feature and silently broke two others. How do you roll out prompt changes safely?
10. [Added] System prompt vs user prompt: what is the instruction hierarchy, and why does it matter?
11. [Added] Your system prompt has grown to 6k tokens and costs too much. How do you shrink it without losing quality?
12. [Added] The model follows your instructions inconsistently. How do you debug that?

### 03 — Embeddings & Vector Databases
1. What is an embedding and why do we need it?
2. How does text get converted into a vector?
3. How do you choose an embedding model? How does dimension affect quality, storage and latency, and which dimension would you pick?
4. How does similarity search work (cosine/dot/L2, ANN: HNSW, IVF, PQ)?
5. How do you choose a vector DB? Compare FAISS, Pinecone, Qdrant, Chroma, Weaviate and pgvector.
6. What would you use for local development vs production, and why (scalability, availability, performance, cost, security, backup, maintenance)?
7. Build a vector store from scratch with no Chroma or Pinecone. What do you implement?
8. How do you detect an embedding model version mismatch?
9. [Added] How do you migrate a live index to a new embedding model with zero downtime?
10. [Added] Metadata filtering with ANN: what goes wrong with pre-filtering vs post-filtering?
11. [Added] Retrieval is poor on domain jargon. Do you fine-tune the embedding model?
12. [Added] Vector storage cost is exploding. What are your options?

### 04 — RAG Core Pipeline
1. Explain an end-to-end RAG pipeline (indexing phase vs retrieval + generation phase) and every component.
2. Why is RAG not just "vector DB + LLM"?
3. Scenario-based chunking: which strategy for unstructured text, structured data, and PDFs with images, tables and text? How does chunk size affect retrieval?
4. How much chunk overlap is enough?
5. How do you choose Top-K?
6. Dense vs sparse retrieval; semantic vs keyword vs hybrid. When can keyword search beat hybrid?
7. How is the BM25 score calculated?
8. BM25 returns n chunks and semantic search returns n chunks. How do you combine them? Explain Reciprocal Rank Fusion.
9. If vector search already returns the top 10, why add a reranker (e.g. BGE)? When should you add one?
10. What roles do embeddings, similarity scores, metadata filters and reranking play in deciding which document is relevant?
11. The LLM is getting irrelevant context. How do you improve retrieval?
12. How do you test for "lost in the middle"?
13. Design citations and source grounding down to document, page, paragraph and line.
14. How do you handle large tables that span multiple pages? What are effective table parsing and ingestion strategies?
15. How do you extract text from scanned PDFs, images, charts and other formats, including with multimodal models?
16. When should the system use RAG instead of relying on model knowledge?
17. Query understanding: query rewriting, expansion and multi-query.
18. [Added] How do you handle conversational follow-ups ("what about for contractors?") in RAG?

### 05 — Advanced RAG
1. Give an overview of advanced RAG: Multi-Query, RAG Fusion, HyDE, Self-RAG, CRAG, Agentic RAG, GraphRAG. What problem does each solve?
2. How does RAG Fusion affect latency?
3. How would you design a cost-efficient CRAG grader?
4. Self-RAG vs CRAG: what are the production cost implications?
5. How would you implement Contextual Retrieval for 10M documents?
6. GraphRAG vs standard RAG: when does graph-based retrieval beat vector search?
7. How do you decide when a multi-hop RAG process should stop?
8. How would you A/B test HyDE?
9. What iteration limits would you use for Agentic RAG?
10. Will integrating agents and tools into a RAG system improve it? Why, and how?
11. [Added] With 1M-token context windows, do we still need RAG?
12. [Added] Parent-document / small-to-big / sentence-window retrieval: when to use them?

### 06 — RAG in Production & Data Freshness
1. The knowledge base updates in real time. How do you avoid stale embeddings and stop stale documents being returned?
2. Document versioning: how do you handle section-level edits and complete section removal while keeping the index consistent?
3. How would you safely handle a failed re-indexing?
4. Design ingestion for 50M documents.
5. How do you handle changed document permissions?
6. Design multi-tenant RAG where tenants must never see each other's data.
7. Where does most RAG latency come from, and how do you reduce it?
8. [Added] Near-duplicate and conflicting documents are polluting retrieval. What do you do?
9. [Added] What does the production layer around RAG look like (caching, re-indexing jobs, cost/latency tracking, PII)?

### 07 — RAG Debugging & Hallucinations
1. RAG accuracy dropped from 85% to 60% after adding documents. How do you find the root cause systematically?
2. The system retrieved the correct document but still gave the wrong answer. Where do you debug first?
3. The chatbot started hallucinating. What do you check, and in what order?
4. How do you reduce hallucination and make sure users never get made-up answers, including making the model refuse?
5. Real scenario: a voice agent invented eight facts in a row even though retrieval returned the right chunks at 0.85–0.9 similarity. How would you debug it?
6. The final answer is wrong. How do you tell a retrieval failure from a generation failure?
7. Faithfulness improves but answer relevancy drops. Why?
8. Faithfulness is 0.95 but users still report wrong answers. What could be wrong?
9. "Your RAG isn't slow because of the LLM; your pipeline is broken." How do you find the slow stage?
10. How do you handle context-window overflow?
11. [Added] Answers were correct yesterday and wrong today, with no code change. What happened?
12. [Added] Users complain answers are "correct but too generic". What do you change?

### 08 — Evaluation
1. How would you measure whether your RAG system actually improved?
2. Explain retrieval metrics: Precision@K, Recall@K, MRR, Hit Rate, nDCG. How do you measure retriever accuracy?
3. Explain RAGAS: faithfulness, answer relevancy, context precision, context recall. How is each computed, and how do you use RAGAS on a response?
4. How do you measure the accuracy of the LLM's generation, and with which metrics?
5. LLM-as-a-Judge: how do you design it, pick representative examples and calibrate against humans?
6. How do you detect verbosity bias (and position bias and self-preference) in an LLM judge?
7. How do you detect eval gaming, where a model overfits to the judge?
8. Design an eval system that catches silent regressions across dozens of prompts and features.
9. How do you run continuous evaluation against a live, shifting production distribution?
10. How many samples do you need before you trust an eval number?
11. How do you evaluate a fine-tuned LLM against its base model?
12. How would you evaluate multimodal RAG beyond RAGAS?
13. How would you monitor hallucination rate automatically?
14. Given user feedback, how do you improve the system?
15. How do you build a ground-truth dataset, and where does human evaluation fit?
16. How do you evaluate agents: tool selection, the planner, and multi-agent systems at the agent, routing, orchestration and end-to-end levels?
17. [Added] Offline metrics improved but online CSAT didn't move. Why?

### 09 — Agents: Fundamentals & Tool Calling
1. What makes a system truly agentic? Chatbot vs agent vs workflow.
2. Walk through agentic architecture: agent core, planning, memory, tools, execution, observation, re-planning.
3. How does function/tool calling actually work under the hood between the LLM and your runtime?
4. How does the LLM decide which tool to call, and why do tool descriptions matter so much?
5. How do you define, register and execute tools? Tools vs functions, schemas and arguments.
6. How would you design a safe tool schema?
7. What decides when an agent stops and returns a final answer?
8. Single-step vs multi-step planning agents: how does an agent decompose a complex task?
9. What is the ReAct pattern, and why interleave reasoning and actions? ReAct vs Plan-and-Execute.
10. How do you handle a plan that must change mid-execution based on a tool result?
11. Write the ReAct loop yourself, with no LangChain.
12. When should you NOT use an AI agent?
13. Design an agent that decides whether to query SQL, a vector DB, an API or the web.
14. The agent produced valid JSON but chose the wrong tool. How do you handle wrong tool selection?
15. Walk-through: "Plan a 3-day trip from Chennai to Coimbatore within ₹10,000." How does the agent handle the constraints?
16. What makes a production agent more than an LLM (state, tools, validation, auth, observability)?

### 10 — Agent Reliability in Production
1. How do you handle a tool call that fails or returns malformed output?
2. How do you validate structured output before acting on it?
3. The agent is stuck calling the same tool. How do you stop it safely and prevent infinite loops and excessive tool calls?
4. How do you handle tool timeouts, retries and rate limits?
5. How do you design retries that don't cause duplicate side effects? How do you make tool execution idempotent?
6. How do you handle partial failures and recover without restarting the whole workflow?
7. Design retry, timeout, fallback and recovery mechanisms for agents.
8. How do you handle long-running tasks (hours or days) with checkpointing, including recovery after a worker crashes halfway?
9. How would you design human-in-the-loop approval for high-risk actions?
10. How do you control cost and set hard token-spend limits when an agent can call tools repeatedly?
11. How do you monitor an agent in production to catch failures before users do?
12. What happens to cost and latency at 10x current usage?
13. An agent gave a wrong answer. How do you identify the cause and correct it?
14. How do you version and roll back an agent's behavior without redeploying the whole system?
15. How do you pause an agent mid-execution and resume it later with the same state?

### 11 — Memory & Context Management
1. Short-term vs long-term memory; working state vs semantic vs episodic memory.
2. What types of memory management exist in RAG and chat systems, and what are the best practices?
3. What should an agent remember, and what should it deliberately forget?
4. How do you stop memory growing unbounded across a long session?
5. How do you handle long conversation history and context-window limits?
6. When do you summarize past context instead of storing it in full?
7. "Summarizing history with an LLM costs money too. Isn't that counterproductive?"
8. How would you architect real-time conversation summarization?
9. How do you maintain context in a long-running chatbot?
10. How do you handle conflicting information in memory?
11. What are the common challenges of memory in long-running agentic systems?
12. [Added] A user says "forget everything about me". How does your memory design support that (privacy/GDPR)?

### 12 — Multi-Agent Systems
1. When is multi-agent actually justified over a single well-designed agent?
2. Design a production-grade multi-agent architecture.
3. How do you define clear responsibilities and boundaries between agents?
4. How do agents communicate, hand off work and share state? How does one agent's output become another's input?
5. Justify your choice of agent-to-agent communication protocol.
6. What is the planner-executor pattern, and when do you need it?
7. Supervisor-worker vs peer-to-peer: what are the trade-offs?
8. How do you stop agents producing duplicate, conflicting or redundant work?
9. How do you coordinate parallel agents without race conditions? Sequential vs concurrent.
10. Design a multi-agent system with explicit failure isolation.
11. What are the common production failure modes of multi-agent systems?
12. How do you debug a failure when it's unclear which agent caused it?
13. Build two agents that review each other's code until it's good. What stops them?
14. Design a Researcher → Analyst → Writer pipeline. What breaks?

### 13 — Agent Frameworks & MCP
1. What does an agent framework give you that raw API calls don't? LangChain (components) vs LangGraph (orchestration).
2. Graph-based (LangGraph) vs role-based (CrewAI): what's the difference?
3. When does a framework add unnecessary abstraction? When would you build a custom orchestration layer?
4. How does a framework track state, branch conditionally and decide which node runs next? Linear chain vs graph.
5. How do you pause an agent mid-execution and resume it with the same state in a framework?
6. How does a framework register and expose tools? How do you handle an unsupported tool, validate tool-call output, and add a custom retry policy to one tool?
7. How do you set hard iteration limits, handle step timeouts and errors, and add HITL before a specific step?
8. How do you version and roll back a workflow definition, not just its prompts?
9. LangGraph vs CrewAI vs Google ADK vs Claude Agent SDK: how do you choose?
10. How do you evaluate whether a framework will scale with your team, not just your prototype?
11. What happens when the framework's abstractions don't match your business logic?
12. Write a working LangGraph summarizer agent.
13. What is MCP, why does it exist, and how is it structured (host/client/server; tools/resources/prompts)?
14. MCP vs API vs plain function calling.
15. "What changed between MCP V1 and V2?"
16. [Added] What are MCP's security risks (tool poisoning, over-permissioned servers, auth)?

### 14 — Fine-Tuning, Model Selection & Optimization
1. RAG or fine-tuning: which is better? When do you choose each?
2. When are prompting and RAG not enough, so that fine-tuning is justified?
3. Explain LoRA/QLoRA, the data curation pipeline, and how you monitor overfitting vs generalization.
4. When would you use a smaller fine-tuned model instead of a frontier model API?
5. Explain distillation, quantization and pruning. When does each degrade quality unacceptably?
6. Open-weight vs closed models in a regulated industry.
7. Design a routing layer that picks between multiple models per request.
8. [Added] What is catastrophic forgetting, and how do you prevent it?
9. [Added] How much data do you need for fine-tuning, and when is synthetic data OK?
10. [Added] SFT vs DPO vs RLHF: which would you use for a behavior problem?

### 15 — LLM Inference & Serving
1. Walk through how you built an LLM inference microservice.
2. What inference optimization techniques do you know (continuous batching, PagedAttention, prefix caching, speculative decoding, FlashAttention, quantization)?
3. How does the KV cache work, and how is it managed (memory math, paging, eviction)?
4. Explain serving parameters: max number of sequences, max batched tokens, GPU memory utilization, max model length.
5. How do you set up HPA for an LLM service and choose its scaling signals?
6. Which latency metrics do you measure (TTFT, TPOT/ITL, E2E, throughput), and why?
7. How do you monitor P50/P95/P99 latency?
8. Which frameworks do you use for LLM deployment (vLLM, TGI, TensorRT-LLM, SGLang, Triton, Ollama)?
9. Stream tokens live over SSE. SSE vs WebSocket, and how do you scale WebSockets?
10. Size the hardware and cost for 10,000 real users.
11. The model doesn't fit your GPU. What are your options (GPU constraints)?
12. Containerize and deploy to a public URL: what does production-ready packaging look like?
13. [Added] Design a voice agent that must reply within ~1–1.2 s. What is the latency budget?

### 16 — Production System Design
1. How do you handle latency, cost, token usage and failures in an LLM application?
2. How would you reduce cost without sacrificing answer quality?
3. What becomes the bottleneck when usage grows 100x?
4. Which types of cache exist (exact, semantic, prompt/prefix, retrieval, embedding), and which fits which scenario?
5. How do you monitor an AI application in production (tracing, logging, dashboards, prompt drift, token usage)?
6. How do you detect model drift and data drift?
7. How do you attribute cost across teams sharing one LLM gateway?
8. Design a multi-region deployment with model failover and data-residency constraints.
9. Design graceful degradation for when the primary provider has a partial outage.
10. Migrate a production system from one model provider to another with zero downtime.
11. [Added] How do you handle provider rate limits at scale?
12. [Added] System design: an enterprise knowledge assistant for 50k employees.
13. [Added] System design: a customer-support AI with human escalation.

### 17 — Security, Safety & Guardrails
1. What is prompt injection (direct and indirect), and how do you defend against it?
2. How do you protect an enterprise RAG system?
3. How do you prevent data leakage and sensitive/PII exposure?
4. What are guardrails? Input vs output validation and the types of guardrails.
5. How do you implement authentication and authorization (OAuth2, RBAC, tool-level permissions and tool authorization)?
6. How do you handle untrusted content the agent reads from a tool result?
7. How do you stop an agent taking a destructive or irreversible action by mistake?
8. Would you let an agent execute code automatically, or require human approval? When?
9. How do you design audit logs and compliance for AI systems?
10. How did you handle security and data privacy in your project?
11. [Added] Jailbreak vs prompt injection: what's the difference?
12. [Added] What happens to data you send to a third-party LLM provider?
13. [Added] How do you red-team an LLM application?

### 18 — NL-to-SQL & Structured Data
1. The generated SQL is syntactically correct but the business answer is wrong. How do you catch that before the user sees it?
2. Design a production NL-to-SQL system.
3. How do you make LLM-generated SQL safe?
4. How do you evaluate NL-to-SQL?
5. [Added] The schema has 1,000+ tables. How do you do schema linking?
6. [Added] When do you route to SQL vs vector search vs both?

### 19 — Classical ML Fundamentals
1. Classification vs regression.
2. Why can accuracy be misleading?
3. Which metrics do you use for imbalanced datasets (precision, recall, F1, ROC-AUC vs PR-AUC)?
4. How do you handle an imbalanced dataset?
5. Micro F1 vs macro F1 (and weighted F1).
6. Neural network basics: loss functions and gradient descent.
7. [Added] Bias-variance trade-off, overfitting and regularization.
8. [Added] What is data leakage, and how do you catch it?
9. [Added] Cross-validation: when and which kind?

### 20 — Python, Backend & Coding
1. When would you use asynchronous programming, and when not?
2. What are the benefits of FastAPI?
3. How do you create a database connection string, and how do you manage secrets?
4. How do you handle database connections for many users? What is SQLAlchemy's connection pool?
5. [Added] GIL, threads vs processes vs asyncio for LLM workloads.
6. [Added] Code: call an LLM concurrently with a rate limit and retries with backoff.
7. [Added] Code: stream an LLM response from FastAPI.
8. Coding: Maximum Subarray Sum (Kadane's algorithm).
9. Coding: implement Reciprocal Rank Fusion.
10. Coding: implement a chunker with overlap and a cosine top-k search.
11. [Added] HTTP fundamentals for AI engineers: status codes (429, 5xx), idempotency keys, timeouts.
12. [Added] What is Pydantic's role in GenAI apps?

### 21 — Domain Scenarios: AI in Supply Chain
1. The AI recommends increasing inventory 30% but Finance rejects it. How do you validate the model?
2. The AI predicts a demand spike that planners don't believe. Model or human experience?
3. An AI agent auto-places purchase orders. What controls do you put around autonomous procurement?
4. Forecast accuracy improved but inventory didn't. What explains the gap?
5. The AI recommends a cheaper but less reliable supplier. How do you evaluate that?
6. The plan is mathematically optimal but causes operational problems. What do you do?
7. The model performs well in one region and poorly in another. How do you investigate?
8. Which supply-chain decisions should never be fully delegated to an AI agent?
9. How do you calculate ROI beyond "hours saved"?
10. The AI's recommendation conflicts with business strategy. Who decides?

### 22 — Strategy, Leadership & Project Deep-Dive
1. Explain your GenAI project architecture end to end.
2. What challenges did you face moving from POC to production?
3. How do you decide build vs buy for a core AI capability?
4. How would you structure a team around model, data and platform ownership?
5. How do you communicate AI risk and limitations to non-technical leadership?
6. Defend a decision: why did you choose it, what alternatives did you consider, how did you prove it works, what changes at scale, what trade-offs did you accept?
7. What should you ask the interviewer? ("How would you answer one of these about your own system?")
8. [Added] Tell me about an AI system failure you owned.
9. [Added] A stakeholder demands 100% accuracy. How do you respond?
10. [Added] How do you keep up with the field without chasing hype?

---

## Suggested 30-day prep (build first, theory when it breaks)

| Week | Build | Topic files |
|---|---|---|
| 1 — Foundations | Tokenize text by hand, prove the "strawberry" problem, build a vector store from scratch | 01, 02, 03, 19, 20 |
| 2 — Retrieval | Ship a full RAG pipeline from chunking to citations; fuse BM25 + vectors with RRF; add a reranker; eval with RAGAS | 04, 05, 06, 07, 08 |
| 3 — Agents | Write a ReAct loop without LangChain; two agents that review each other's code; a LangGraph summarizer; an MCP server | 09, 10, 11, 12, 13 |
| 4 — Production | Stream tokens over SSE; size hardware and cost for 10k users; containerize and deploy; add guardrails and tracing | 14, 15, 16, 17, 18, 21, 22 |

**Daily:** one build, three interview questions, one technical problem, one system-design pattern.
