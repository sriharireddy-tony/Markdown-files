# 12 — Multi-Agent Systems

**What interviewers probe here:** whether you can justify multiple agents at all, and whether you design them like a distributed system: clear contracts, isolation, coordination, idempotency and observability. The senior signal is knowing when *not* to go multi-agent.

[← Back to index](README.md)

1. [When is multi-agent justified over a single agent?](#q1-when-is-multi-agent-actually-justified-over-a-single-well-designed-agent)
2. [Design a production-grade multi-agent architecture](#q2-design-a-production-grade-multi-agent-architecture)
3. [Responsibilities and boundaries between agents](#q3-how-do-you-define-clear-responsibilities-and-boundaries-between-agents)
4. [Communication, handoffs and shared state](#q4-how-do-agents-communicate-hand-off-work-and-share-state-how-does-one-agents-output-become-anothers-input)
5. [Agent-to-agent communication protocol](#q5-justify-your-choice-of-agent-to-agent-communication-protocol)
6. [Planner-executor pattern](#q6-what-is-the-planner-executor-pattern-and-when-do-you-need-it)
7. [Supervisor-worker vs peer-to-peer](#q7-supervisor-worker-vs-peer-to-peer-what-are-the-trade-offs)
8. [Preventing duplicate, conflicting or redundant work](#q8-how-do-you-stop-agents-producing-duplicate-conflicting-or-redundant-work)
9. [Parallel agents without race conditions](#q9-how-do-you-coordinate-parallel-agents-without-race-conditions-sequential-vs-concurrent)
10. [Failure isolation](#q10-design-a-multi-agent-system-with-explicit-failure-isolation)
11. [Common production failure modes](#q11-what-are-the-common-production-failure-modes-of-multi-agent-systems)
12. [Debugging which agent caused a failure](#q12-how-do-you-debug-a-failure-when-its-unclear-which-agent-caused-it)
13. [Two agents reviewing each other's code](#q13-build-two-agents-that-review-each-others-code-until-its-good-what-stops-them)
14. [Researcher → Analyst → Writer pipeline](#q14-design-a-researcher--analyst--writer-pipeline-what-breaks)

---

## Q1. When is multi-agent actually justified over a single well-designed agent?
*Source: interview posts*

**What the interviewer is testing:** Restraint. Can you resist architecture for its own sake?

**Strong answer:**

My default is **one agent with good tools**, and I only split when there's a concrete reason. Every extra agent adds LLM calls (cost), latency, handoff errors and debugging complexity.

Multi-agent is justified when:
1. **Context isolation is needed.** Subtasks each need lots of context that would pollute one window. For example, 5 sub-agents each read 50 pages and return 1-page findings, so the orchestrator never sees 250 pages.
2. **Real parallelism exists.** Independent subtasks (research 6 competitors) can run concurrently and cut wall-clock time.
3. **Different permissions or trust levels are needed.** A "reader" agent that touches untrusted web content has no write tools. An "executor" with write tools never sees raw web content. That's a security boundary.
4. **Different models or skills fit different steps.** A cheap model for triage and a strong model for reasoning.
5. **Too many tools.** One agent with 60 tools selects poorly. Grouping into specialists with about 8 tools each improves selection accuracy.
6. **Organizational ownership.** Different teams own different agents with separate release cycles.

It's *not* justified when the task is sequential and simple, when "agents" are really just prompt templates in a fixed pipeline (that's a workflow), or when latency budgets are tight (a voice bot with a 1 s budget can't afford agent handoffs).

My decision test: **"Can I get the same result with one agent plus better tools and prompts?"** Build that first, measure, and split only the part that fails.

Example: a due-diligence assistant. The single agent hit 180k tokens and lost details. Splitting into an orchestrator plus parallel "document analyst" sub-agents (each returning a structured summary) cut orchestrator context to 20k, reduced wall-clock from 9 to 3 minutes, and improved the fact-recall eval from 72% to 88%, at about 2.5x total token cost, which the business accepted.

**Follow-ups:**
- *Cost impact?* → Multi-agent research-style systems commonly use several times more tokens than single-agent chat. It needs a clear quality or latency win to justify.
- *How do you decide the split?* → Along context boundaries, permission boundaries or parallelism, not along job titles.

**Red flags:**
- "Multi-agent is always better because each agent specializes."
- Splitting into CEO/manager/worker roles with no measurable reason.

---

## Q2. Design a production-grade multi-agent architecture.
*Source: interview posts*

**What the interviewer is testing:** A complete system view: orchestration, state, tools, guardrails, observability and failure handling.

**Strong answer:**

I'd clarify the use case first. Say it's an "enterprise request handler" that answers questions and executes HR/IT actions.

```
            ┌──────────── API Gateway (auth, rate limit, tenant) ───────────┐
            ▼                                                               │
     Orchestrator / Supervisor (LLM + deterministic router)                 │
       │  typed task messages          ▲ typed results                      │
       ├──► Retrieval agent (RAG, read-only tools)                          │
       ├──► Data agent (SQL, read-only, row-level security)                 │
       ├──► Action agent (write tools, HITL for high-risk)                  │
       └──► Critic/verifier (checks grounding, policy)                      │
                                                                            │
 Shared infra: State store + checkpointer (Postgres) │ Task queue (retries,  │
 DLQ) │ Tool gateway (authZ, idempotency keys, rate limits) │ Guardrails     │
 (input/output) │ Tracing (OpenTelemetry/LangSmith, one trace_id per request)│
 Budgets (max steps, tokens, $ per request) │ Prompt/agent registry (versioned)
```

Key design decisions:
- **Orchestration style:** a supervisor with a mostly deterministic graph (LangGraph-style). An LLM decides routing only where it adds value, and code enforces the allowed transitions.
- **Contracts:** each agent has a typed input/output schema (Pydantic). Agents exchange **structured results**, not chat transcripts.
- **State:** a central, checkpointed state per request. Agents read the slices they need and write via reducers.
- **Tools behind a gateway:** authorization uses the *end user's* identity, and every write gets an idempotency key, audit logging and rate limits.
- **Safety:** least-privilege tool sets per agent, and human approval for irreversible actions.
- **Budgets:** global step limit (e.g. 25), per-agent limits, a token/$ ceiling, and wall-clock timeouts.
- **Failure handling:** per-agent timeouts and retries, fallback responses, circuit breakers, and resume from checkpoint.
- **Versioning:** agents, prompts and graph versions in a registry, with canary rollout.
- **Evaluation:** component-level (routing accuracy, tool-selection accuracy) plus end-to-end task success.

Scale numbers: 20k requests/day, P50 latency 6 s and P95 18 s, average 4.2 LLM calls per request, and about $0.02 per request with a cheap router and a strong model only for synthesis.

**Follow-ups:**
- *Why not let agents talk freely?* → Unbounded conversations are expensive, non-deterministic and hard to debug. Constrained topology is easier to evaluate.
- *How do you deploy it?* → Each agent is a stateless worker behind the queue, and they scale independently.

**Red flags:**
- A diagram of agents with no state, budgets, auth or observability.
- Agents passing entire chat histories to each other.

---

## Q3. How do you define clear responsibilities and boundaries between agents?
*Source: interview posts*

**What the interviewer is testing:** Applying software-design principles (single responsibility, interfaces) to agents.

**Strong answer:**

I define each agent like a microservice, with a **contract**:

| Contract element | Example: "Invoice Data Agent" |
|---|---|
| **Purpose (one sentence)** | Answer questions about invoice data for the requesting tenant |
| **Inputs (schema)** | `{question: str, tenant_id: str, date_range?: DateRange}` |
| **Outputs (schema)** | `{answer: str, sql: str, rows: int, confidence: float, sources: []}` |
| **Tools allowed** | `run_readonly_sql`, `get_schema` only |
| **Out of scope** | Anything that writes; questions outside invoices, which it returns `OUT_OF_SCOPE` for |
| **Budget** | Max 6 steps, 20k tokens, 30 s |
| **Owner** | Finance platform team |

Principles:
- **Single responsibility**: one domain or capability per agent. If its description needs "and" three times, split it or simplify it.
- **Non-overlapping tool sets.** If two agents can both send emails, you'll get duplicate emails. Each side effect has exactly one owning agent.
- **Explicit "I can't do this" output.** Agents return a typed refusal or escalation instead of improvising outside scope.
- **Router descriptions are the boundary.** The supervisor routes based on agent descriptions, so write them like API docs, including negative examples ("do NOT use for payroll questions").
- **Test the boundary.** Evals with out-of-scope inputs verify that the agent declines and the router re-routes.

Example: we had a "Support agent" and a "Billing agent" that both answered refund questions and gave different policies. We fixed it by making Billing the sole owner of refunds, adding "refunds → Billing" to the router rules, and removing the refund tool from Support. Conflicting answers dropped to near zero.

**Follow-ups:**
- *How many tools per agent?* → Keep it small, roughly 5–15. Selection accuracy drops as tool count and description overlap grow.
- *Who owns shared concerns like logging?* → Infrastructure, not agents: gateway, middleware, guardrails.

**Red flags:**
- Boundaries defined only by persona prompts ("You are a senior analyst").
- Overlapping tools with side effects.

---

## Q4. How do agents communicate, hand off work and share state? How does one agent's output become another's input?
*Source: interview posts*

**What the interviewer is testing:** Understanding of the communication patterns and why structured handoffs beat free text.

**Strong answer:**

Three main patterns:

1. **Shared state (blackboard).** All agents read and write a central typed state object. This is LangGraph's model: nodes return partial updates, and reducers merge them (e.g. `messages` appended, `findings` merged by key). Good for a single process or workflow with checkpointing.
2. **Message passing.** Agents send typed messages via direct calls or a queue (e.g. `TaskRequest{task_id, goal, inputs, constraints}` → `TaskResult{task_id, status, output, artifacts, errors}`). Good for distributed agents, different services or teams, and async work.
3. **Handoff / transfer of control.** One agent transfers the conversation to another, as in the OpenAI Agents SDK handoffs or LangGraph `Command(goto=...)`. Good for customer-facing flows ("transferring you to billing").

How output becomes input, done well:
- **Structured, not narrative.** The researcher returns `{claims: [{text, source_url, confidence}]}`, not a 3-page essay the next agent has to re-parse.
- **Validate at the boundary.** Pydantic validation on every handoff. If it's invalid, return to the producer with the error (bounded retries).
- **Pass references for large artifacts.** `document_id` or an S3 URI instead of the content.
- **Pass only what's needed.** The writer agent gets the findings plus the style guide, not the researcher's full tool trace. This is context isolation.
- **Carry correlation IDs.** `trace_id`, `task_id` and `parent_task_id` on every message, for debugging and idempotency.

```
Researcher ──TaskResult{claims[], sources[]}──► [schema validation] ──► Analyst
                                   invalid ──► back to Researcher (max 2 retries)
```

Example: our analyst agent hallucinated numbers because the researcher's prose output mixed "I think" with sourced facts. Switching to a `claims[]` schema with a mandatory `source_url` meant the analyst could only use sourced claims, and unsupported numbers in the final report dropped by about 80%.

**Follow-ups:**
- *Shared state vs messages, which one?* → Shared state within one workflow runtime, messages across services or teams.
- *How do you handle schema evolution?* → Version the message schemas and keep them backward compatible, like APIs.

**Red flags:**
- "Agents just chat with each other in natural language."
- No validation between agents.

---

## Q5. Justify your choice of agent-to-agent communication protocol.
*Source: interview posts*

**What the interviewer is testing:** Awareness of the protocol landscape (MCP, A2A, plain RPC and queues) and the ability to pick based on constraints.

**Strong answer:**

First I'd separate two different problems:
- **Agent-to-tool/context:** MCP (Model Context Protocol) standardizes how an agent host connects to tools, resources and prompts exposed by servers.
- **Agent-to-agent:** A2A (the Agent2Agent protocol, originally from Google, now under the Linux Foundation) standardizes how independent agents discover each other (Agent Cards), send tasks, stream progress, and return artifacts over HTTP/JSON-RPC.

My choice depends on the deployment boundary:

| Situation | Choice | Why |
|---|---|---|
| All agents in one service, one team | In-process calls with a shared typed state (LangGraph nodes/subgraphs) | Lowest latency, easiest checkpointing and tracing |
| Agents as internal services, same org | Typed RPC (HTTP/gRPC) or a queue (SQS/Kafka) with versioned schemas | Mature infra: retries, DLQs, auth, monitoring |
| Long-running async tasks | Queue + task-status store (or A2A tasks with streaming updates) | Decoupling, backpressure, resumability |
| Agents across teams, vendors or orgs | A2A | Standard discovery, task lifecycle, and interop across frameworks |
| Exposing capabilities to many agent hosts | MCP server | Any MCP-compatible client can use it |

What I'd justify in any protocol choice:
- **Typed contracts** and schema versioning.
- **Task lifecycle** (submitted → working → input-required → completed/failed) for long tasks.
- **Auth** between agents: service identity plus propagated user identity, least privilege.
- **Idempotency and correlation IDs.**
- **Observability**: trace-context propagation across hops.

Example answer: "For our internal system I chose in-process LangGraph subgraphs for the core flow, because it's one team, sub-second handoffs and shared checkpoints. We expose our HR data agent to other business units via A2A so their agents, built on different frameworks, can delegate tasks without custom integrations. Tools are exposed through MCP servers."

**Follow-ups:**
- *Is A2A a replacement for MCP?* → No. They're complementary: MCP connects agent to tools, A2A connects agent to agent.
- *Why not use A2A for everything?* → Network hops, serialization and auth overhead are unnecessary inside one process.

**Red flags:**
- Confusing MCP with agent-to-agent communication.
- Picking a protocol because it's trendy, without deployment reasons.

---

## Q6. What is the planner-executor pattern, and when do you need it?
*Source: interview posts*

**What the interviewer is testing:** Understanding of plan-then-act, its benefits over pure ReAct, and when it fails.

**Strong answer:**

Planner-executor separates **deciding what to do** from **doing it**:

```
Goal ──► Planner (strong model) ──► Plan: [step1, step2, step3 ...] (structured)
                                       │
               ┌───────────────────────┘
               ▼
        Executor(s) (cheaper model / deterministic code) run each step with tools
               │ results
               ▼
        Re-planner: done? adjust remaining steps? ──► final answer
```

Why use it:
- **Cost:** the expensive model plans once, and cheap models or plain code execute the steps. A step like "fetch invoices for Q3" might not need an LLM at all.
- **Parallelism:** independent steps in the plan (a DAG) can run concurrently.
- **Reviewability:** the plan can be shown to a human for approval *before* any action. This is great for high-risk workflows.
- **Coherence on long tasks:** an explicit plan reduces wandering compared with step-by-step ReAct.

When you need it:
- Multi-step tasks with a predictable structure (reports, migrations, data pipelines, research).
- When you want HITL approval of the plan.
- When steps are parallelizable.

When not to use it:
- Highly exploratory tasks where the next step depends heavily on what you just found (debugging, open web research). ReAct adapts better there.
- Simple tasks: planning overhead exceeds the benefit.

Failure modes and mitigations: plans go stale when reality differs, so **re-plan after each step or on failure**. Planners may produce steps the executor can't do, so give the planner the list of executor capabilities and validate the plan schema. Plans also shouldn't be unbounded, so cap the step count.

Example: a quarterly-report agent. The planner (strong model) produced a 7-step DAG. Four data-fetch steps ran in parallel as code, and the two analysis steps used a mid-tier model. Latency went from 95 s (ReAct) to 38 s and cost dropped about 55%, with equal quality on our eval.

**Follow-ups:**
- *Planner-executor vs supervisor?* → A supervisor routes dynamically each turn. A planner commits to a plan upfront and revises it.
- *What does re-planning look like?* → Pass the plan, completed steps and results, and the failure to the planner. It returns the updated remaining steps.

**Red flags:**
- No re-planning. "The plan is executed as-is."
- Using an LLM for executor steps that are deterministic.

---

## Q7. Supervisor-worker vs peer-to-peer: what are the trade-offs?
*Source: interview posts*

**What the interviewer is testing:** Topology trade-offs: control, latency, bottlenecks, emergence.

**Strong answer:**

```
Supervisor-worker (hub & spoke)        Peer-to-peer (mesh / swarm)
        [Supervisor]                      [A] ⇄ [B]
       /     |      \                      ⇅  ✕  ⇅
    [W1]   [W2]    [W3]                   [C] ⇄ [D]
```

| Dimension | Supervisor-worker | Peer-to-peer |
|---|---|---|
| Control and predictability | High: one place decides routing | Low: emergent flows |
| Debuggability | Easy: one decision log | Hard: tracing across many edges |
| Latency | Extra hop through the supervisor on every step | Direct handoffs can be faster |
| Bottleneck / SPOF | Supervisor context grows; single point of failure | No central bottleneck |
| Cost | Supervisor tokens on every step | Fewer coordinator tokens, but risk of chatter loops |
| Loops and termination | Easy to enforce limits centrally | Needs a distributed stop condition, TTL/hop counts |
| Best for | Enterprise workflows, compliance, customer-facing | Exploratory, creative, research simulations; loosely coupled cross-org agents |

In practice I choose **supervisor-worker for production**, and often **hierarchical** at scale: a top supervisor over team supervisors, each with a few workers. That keeps each supervisor's context and tool list small.

A useful middle ground is **handoffs**: agents transfer control directly to a named peer, but only along **allowed edges** defined in the graph. That gives peer-style latency with supervisor-style control.

Mitigating supervisor weaknesses: keep the supervisor's context small (workers return summaries), make routing partially deterministic (rules first, LLM for ambiguous cases), and checkpoint so a supervisor crash can resume.

Example: a support system started peer-to-peer (triage ⇄ billing ⇄ tech). About 6% of conversations bounced between agents 4+ times. Moving to a supervisor with a max of 2 transfers and an escalation-to-human rule removed the ping-pong and made routing accuracy measurable (94%).

**Follow-ups:**
- *How do you prevent peer ping-pong?* → A hop counter in the message, a "visited agents" list, and a max-transfers rule.
- *When does the supervisor become the bottleneck?* → When it has too many workers (more than about 8–10) or ingests full worker outputs. Go hierarchical.

**Red flags:**
- "Peer-to-peer is more scalable, so it's better", with no mention of control or debuggability.
- No termination strategy for the mesh.

---

## Q8. How do you stop agents producing duplicate, conflicting or redundant work?
*Source: interview posts*

**What the interviewer is testing:** Coordination mechanisms: ownership, task registry, dedupe, conflict resolution.

**Strong answer:**

These are three different problems, each with its own fix:

**Duplicate work** (two agents do the same thing):
- **Central task registry / plan.** The supervisor or planner assigns each subtask a unique `task_id` and an owner. Workers claim tasks atomically (DB row lock or a queue with visibility timeout), so nobody picks up an already-claimed task.
- **Explicit, disjoint scopes** in delegation: "Agent A: competitors 1–3, Agent B: competitors 4–6", not "research competitors".
- **Shared "done" state.** Record visited URLs, executed queries and completed task IDs in state and check them before acting.
- **Idempotency keys on side-effecting tools**, so even if duplication happens, the email or payment executes once.

**Conflicting results** (agents disagree):
- **Single source of truth per fact type**: one owning agent or tool.
- **A reconciliation step**: a verifier or aggregator compares outputs, flags contradictions, and resolves them by evidence (sourced beats unsourced, fresher beats older) or escalates.
- **Structured outputs with sources and confidence** so conflicts are detectable in code.

**Redundant work** (unnecessary steps):
- The planner produces a DAG, and I dedupe near-identical subtasks (embedding similarity on task descriptions).
- Cache tool results by `(tool, args)` within a request, so a second identical search is free.
- Budgets per agent force prioritization.

Example: in a parallel research system, 3 of 5 sub-agents often ran the same top search queries. Adding a shared query cache and disjoint sub-topic assignment from the planner reduced search calls by 45% and tokens by 30%, and the aggregator's contradiction check cut conflicting statements in reports from about 9% to 2%.

**Follow-ups:**
- *What if two agents write to the same record?* → A single writer per resource, or optimistic concurrency with version checks. See Q9.
- *How do you detect conflicts automatically?* → Pairwise NLI or an LLM contradiction check across claims on the same entity.

**Red flags:**
- "The final LLM will merge everything nicely."
- No idempotency for side effects.

---

## Q9. How do you coordinate parallel agents without race conditions? Sequential vs concurrent.
*Source: interview posts*

**What the interviewer is testing:** Concurrency fundamentals applied to agents.

**Strong answer:**

**Sequential vs concurrent: how I decide.** I draw the dependency graph of subtasks.
- **Concurrent** when subtasks are independent (no data dependency), read-only or writing disjoint resources, and latency matters. Example: researching 6 vendors, summarizing 20 documents.
- **Sequential** when step B needs A's output, when steps mutate shared state, when order matters (create account → assign license → send welcome), or when rate limits or cost make fan-out unwise.
- Often it's **mixed**: a DAG with parallel fan-out, then a join (map-reduce).

**Avoiding race conditions:**
1. **Workers don't share mutable state directly.** Each parallel branch writes to its own slot, e.g. `results[task_id]`, and a **reducer merges** at the join. LangGraph's `Send` API plus reducer-annotated state fields (`Annotated[list, operator.add]`) does exactly this.
2. **Single writer per external resource**, or serialize writes through one executor or queue partition keyed by resource ID (Kafka partition by `account_id`).
3. **Optimistic concurrency** for shared records: read with a version, write `WHERE version = n`, and on conflict re-read and retry.
4. **Idempotency keys** so retries after timeouts don't double-apply.
5. **Barriers / joins with timeouts.** The aggregator waits for all branches or a deadline, then proceeds with partial results marked as partial.
6. **Bounded concurrency.** A semaphore on parallel LLM and tool calls (e.g. 10) to respect rate limits. An unbounded fan-out causes 429s, which cause retries, which cause duplicates.

```
          ┌─► worker(task1) ─► results[t1] ┐
plan ─────┼─► worker(task2) ─► results[t2] ┼─► reducer/join (timeout 60s) ─► synthesize
          └─► worker(task3) ─► results[t3] ┘
```

Example: parallel ticket-update agents occasionally overwrote each other's status changes in the ticketing system. We routed all writes through a single "ticket writer" tool with per-ticket locking and version checks, and lost updates went to zero.

**Follow-ups:**
- *What if one branch is slow?* → Deadline at the join, proceed with partial results, and optionally cancel the straggler.
- *Async in Python?* → `asyncio.gather` with a `Semaphore` and per-task timeouts, and `return_exceptions=True` so one failure doesn't cancel the others.

**Red flags:**
- Parallelizing everything "for speed" without dependency analysis.
- Multiple agents appending to the same list or record without a merge strategy.

---

## Q10. Design a multi-agent system with explicit failure isolation.
*Source: interview posts*

**What the interviewer is testing:** Applying fault-tolerance patterns (bulkheads, circuit breakers, timeouts) to agents.

**Strong answer:**

The goal is that **one agent failing degrades one capability, not the whole request or the whole system.**

Isolation layers:
1. **Process and resource isolation (bulkheads).** Each agent type runs as its own worker pool or deployment, with its own concurrency limits and queue. A runaway research agent can't starve the billing agent of workers or rate-limit quota. Rate limits and budgets are also allocated per agent.
2. **Timeouts at every boundary.** Per tool call (e.g. 10 s), per agent task (60 s), per request (120 s). A hung agent is treated as a failure, not waited on forever.
3. **Circuit breakers** per agent and per downstream dependency. If the "flight search" tool fails 50% over 1 minute, open the circuit and fail fast, so the planner skips that capability or uses a fallback.
4. **Typed failure results.** An agent returns `status: failed | partial | success` with an error class, never an exception that bubbles up and kills the orchestrator. The supervisor has a policy per failure class: retry, route to a fallback agent, degrade, or escalate.
5. **Blast-radius control on side effects.** Actions are staged (draft → approve → commit) and compensating actions (sagas) are defined. If step 4 fails after step 3 committed, run the compensation for step 3 or flag for human review.
6. **Checkpointing** so a failed agent's step can be retried or resumed without re-running successful agents.
7. **Context isolation.** A poisoned or confused agent's output passes through schema validation and a verifier before entering shared state, so bad data doesn't propagate.

```
Supervisor ── task ──► [Agent pool A]  (queue A, limits A, breaker A)
          └─ task ──► [Agent pool B]  (queue B, limits B, breaker B)
  result: {status, output?, error_class?}  → policy: retry | fallback | degrade | escalate
```

Example: a travel assistant with flight, hotel and visa agents. When the hotel provider API went down, the breaker opened and the hotel agent returned `failed: dependency_unavailable`. The supervisor delivered flights and visa info plus "hotel search is temporarily unavailable, I'll notify you". The request completed with partial results instead of a 500 error.

**Follow-ups:**
- *Retry vs fallback?* → Retry transient errors (timeouts, 429, 503) with backoff and jitter. Fall back on persistent errors or an open circuit.
- *How do you test it?* → Chaos testing: inject tool failures, latency and malformed outputs in staging.

**Red flags:**
- One try/except around the whole workflow.
- No compensation strategy for partially completed side effects.

---

## Q11. What are the common production failure modes of multi-agent systems?
*Source: interview posts*

**What the interviewer is testing:** Real-world experience and a structured taxonomy.

**Strong answer:**

I group them into four buckets:

**Specification and design failures**
- Vague role or task specs lead to agents misunderstanding scope.
- Overlapping responsibilities lead to duplicate or conflicting actions.
- Missing termination criteria lead to endless back-and-forth.

**Coordination failures**
- **Handoff information loss.** The next agent lacks context it needed, or gets too much irrelevant context.
- **Ping-pong / loops** between agents ("over to you", "no, over to you").
- **Agents ignoring others' outputs** or silently overriding them.
- **Race conditions** on shared state or resources.
- **Deadlocks:** A waits for B, which waits for A's clarification.

**Quality and verification failures**
- **Error propagation / compounding.** An upstream hallucination is accepted as fact downstream and amplified.
- **Premature termination.** The agent declares success without verifying.
- **Weak or absent verification**, or a verifier that just agrees ("sycophantic review").

**Operational failures**
- **Cost and latency explosions.** Fan-out multiplies calls, and retries multiply more.
- **Rate-limit cascades.** Parallel agents hit the provider limit, then retry storms.
- **Tool and API failures** not handled per agent.
- **State loss on crash** without checkpointing.
- **Prompt injection spreading.** Untrusted content read by one agent becomes instructions for another.
- **Version skew.** Agent A updated its output schema and agent B wasn't updated.

Mitigations map directly: typed contracts, a supervisor with hop limits, verifiers grounded in evidence, budgets, bulkheads, checkpointing, and end-to-end traces.

Example: in one incident, a "summarizer" agent's new prompt version dropped the `sources` field. The downstream "fact-checker" treated missing sources as "no issues found", so accuracy silently dropped for 2 days. The fix was schema validation at the handoff (missing field means failure, not pass) plus a contract test in CI.

**Follow-ups:**
- *Which is most common?* → In my experience, specification problems and handoff information loss. Most failures are design, not model capability.
- *How do you catch error propagation?* → Require sources per claim and verify claims against sources at the aggregation step.

**Red flags:**
- Listing only "hallucination".
- No operational or security failure modes.

---

## Q12. How do you debug a failure when it's unclear which agent caused it?
*Source: interview posts*

**What the interviewer is testing:** Observability discipline and a systematic root-cause process.

**Strong answer:**

You can't debug multi-agent systems from final outputs. You need **distributed tracing designed in from day one.**

**Instrumentation:**
- One `trace_id` per user request, propagated to every agent, LLM call and tool call as spans (OpenTelemetry, or LangSmith, Langfuse, Arize Phoenix).
- Each span logs: agent name and version, prompt version, model, inputs and outputs (PII-redacted), tool arguments and results, tokens, latency, and the routing decision with its reason.
- Persist state checkpoints per step, so you can inspect the exact state at every handoff.

**Debugging process:**
1. **Reproduce the trace.** Open the failing trace and walk the timeline.
2. **Check the handoffs first.** At each boundary, was the input to agent N correct? The first agent whose *input was correct but output was wrong* is the culprit. This is bisection across the chain.
3. **Classify the failure:** routing (wrong agent picked), tool selection or arguments, tool failure, reasoning or hallucination, handoff information loss, or termination.
4. **Replay in isolation.** Take agent N's exact input from the checkpoint and re-run it alone, multiple times (it's non-deterministic), with different prompt or model versions. LangGraph lets you replay or fork from a checkpoint.
5. **Check for recent changes.** Diff prompt, model and schema versions against the last good run.
6. **Fix and add the case to the regression eval set** at both component and end-to-end level.

**Aggregate analysis** for non-reproducible issues: tag failures, and look for correlations with agent version, tool, input type or step count.

Example: users reported wrong totals in expense reports. The traces showed the extraction agent's output was correct. The aggregator received the right items but double-counted them, because a retry had appended results twice (a list reducer with no dedupe). The fix was to key results by `item_id`. We found it in 20 minutes because each handoff's state was in the checkpoint store.

**Follow-ups:**
- *What if traces contain sensitive data?* → Redact at ingestion, set retention limits, and restrict access by role.
- *LLM-assisted debugging?* → Useful to classify failure types across thousands of traces, but confirm with a human.

**Red flags:**
- "I'd add print statements."
- Re-running the whole pipeline end to end and eyeballing it.

---

## Q13. Build two agents that review each other's code until it's good. What stops them?
*Source: interview posts*

**What the interviewer is testing:** Designing a generator-critic loop with objective termination, not an infinite polite conversation.

**Strong answer:**

It's a **generator-critic (reflection) loop**. The danger is that two LLMs will happily loop forever, or the critic rubber-stamps everything. So "good" must be defined **objectively**, mostly by code rather than by LLM opinion.

```
Task ─► Coder ─► code ─► Gates (deterministic): lint, typecheck, unit tests, security scan
                             │ fail → errors back to Coder
                             │ pass
                             ▼
                          Reviewer (LLM, rubric) ─► {approve | issues[] with severity}
                             │ blocking issues → Coder
                             ▼
                           Done / escalate
```

**What stops them (termination conditions, checked in code):**
1. **Success:** all deterministic gates pass **and** the reviewer returns no issues of severity blocker or major.
2. **Iteration cap:** max 3–5 rounds.
3. **No progress:** the same test failures or the same reviewer issues two rounds in a row, or the diff between versions is trivial. Stop and escalate.
4. **Budget:** a token or $ ceiling per task.
5. **Oscillation detection:** hash each code version. If a version repeats (A→B→A), stop.

**Making the review useful:**
- The reviewer gets a **rubric** (correctness, edge cases, security, readability) and must output **structured issues** with line references and severity, not prose.
- The reviewer sees the test results, so it doesn't re-litigate things tests already verify.
- Use a **different model or prompt** for the reviewer to reduce shared blind spots, and tell it that approving is fine when there are no real issues. Otherwise it invents nitpicks to look useful.
- Only **blocking** issues trigger another round. Nits are listed but don't loop.

Example numbers: on an internal benchmark of 200 tasks, a single-pass coder passed tests on 61%. The loop with test gates reached 83% after a median of 2 rounds. Adding the LLM reviewer on top raised security-issue detection but only +2% on tests. The cap of 4 rounds kept P95 cost at about 3x single-pass.

**Follow-ups:**
- *Why not let the reviewer decide "good"?* → LLM judges are lenient or sycophantic and inconsistent. Tests are objective.
- *What if there are no tests?* → Have a separate agent generate tests from the spec first, and review those tests.

**Red flags:**
- "They keep going until the reviewer says it's good."
- No iteration cap or progress detection.

---

## Q14. Design a Researcher → Analyst → Writer pipeline. What breaks?
*Source: interview posts*

**What the interviewer is testing:** Designing a sequential multi-agent pipeline and anticipating its failure points.

**Strong answer:**

It's really a **workflow with agentic steps**, so I'd build it as a graph with typed handoffs:

```
Brief ─► Researcher ──claims[]──► Analyst ──insights[]──► Writer ──draft──► Verifier ─► Report
         (search, fetch,           (compare, compute,     (outline,          (claim-source
          extract claims            find patterns,         write with         check, style)
          w/ source + quote)        cite claim_ids)        citations)             │ fail → Writer (max 2)
```

- **Researcher:** tools for search and fetch. Outputs `claims: [{id, text, source_url, quote, date, confidence}]`. Parallel sub-researchers per sub-topic, deduped by URL.
- **Analyst:** no web access (context isolation and injection safety). Uses only the claims, plus a code tool for calculations. Outputs `insights: [{text, supporting_claim_ids[]}]`.
- **Writer:** gets insights plus claims plus a style guide and audience. Every sentence with a fact must cite a claim ID.
- **Verifier:** checks each citation against the quoted source text (does the claim support the sentence?) and flags uncited facts.

**What breaks, and the fix:**

| Failure | Fix |
|---|---|
| Researcher returns low-quality or outdated sources | Source quality filters, recency requirements, allow-lists for domains |
| Claims without sources, or fabricated quotes | Require a verbatim quote; verify the quote exists in the fetched text (string match) |
| Information loss at handoff (analyst lacks nuance) | Structured claims include the quote and context; analyst can request more research (bounded) |
| Analyst hallucinates numbers or bad math | Calculations in a code tool, not in the LLM's head |
| Writer "improves" facts or drops citations | Citation-required prompt plus the verifier gate |
| Error compounding from one bad claim | Verifier checks against sources, not against the analyst's text |
| Prompt injection from web pages | Only the researcher touches raw content; it outputs data, never instructions; no tools with side effects |
| Cost and latency blowup | Cap sources (e.g. 25), parallel research, cheap model for extraction, strong model for analysis |
| Endless revise loops | Max 2 verifier rounds, then ship with flagged items for human review |

Example numbers: a market-research report generator produced 8-page reports in about 4 minutes for about $0.60 each. The verifier caught unsupported claims in about 12% of drafts. After the fix loop, the human-audit error rate was 1.5% of claims (from 9% with the naive version).

**Follow-ups:**
- *Why not one agent?* → For short reports, one agent is fine. The split pays off for context isolation (researching 25 sources), injection safety, and targeted evaluation of each stage.
- *How do you evaluate it?* → Per stage: claim precision (sourced and correct), insight support rate, citation accuracy. End to end: human rubric scoring.

**Red flags:**
- Passing prose between agents.
- No verification step; trusting the writer's output.

---

## Rapid-fire recap

- Default to one agent with good tools; split for context isolation, parallelism, permission boundaries or tool overload.
- Define each agent like a microservice: purpose, typed I/O, allowed tools, budget, owner.
- Each side effect has exactly one owning agent.
- Pass structured results and references between agents, never whole transcripts.
- MCP connects agents to tools; A2A connects agents to agents; in-process state is best within one service.
- Planner-executor saves cost and enables plan approval; always re-plan on failure.
- Supervisor-worker for production control; go hierarchical when the supervisor gets overloaded.
- Prevent races with per-branch outputs and reducers, single writers, optimistic locks and idempotency keys.
- Isolate failures with bulkheads, timeouts, circuit breakers and typed failure results.
- The most common failures are specification and handoff problems, not model capability.
- Debug by bisecting handoffs in a distributed trace, then replay the culprit agent from its checkpoint.
- Generator-critic loops stop on objective gates, iteration caps, no-progress detection and budgets.
