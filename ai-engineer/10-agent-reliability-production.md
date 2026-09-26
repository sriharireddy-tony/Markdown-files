# 10 — Agent Reliability in Production

**What interviewers probe here:** can this system be trusted to act on its own, and what stops it when something goes wrong? Expect questions on failure handling, loops, retries without duplicate side effects, long-running durability, human approval, cost control, and versioning agent behavior.

[← Back to index](README.md)

1. [Tool call fails or returns malformed output](#q1)
2. [Validating structured output before acting](#q2)
3. [Stuck calling the same tool / infinite loops](#q3)
4. [Tool timeouts, retries and rate limits](#q4)
5. [Retries without duplicate side effects / idempotency](#q5)
6. [Partial failures and recovering without restarting](#q6)
7. [Retry, timeout, fallback and recovery design](#q7)
8. [Long-running tasks, checkpointing and crash recovery](#q8)
9. [Human-in-the-loop for high-risk actions](#q9)
10. [Controlling cost and token-spend limits](#q10)
11. [Monitoring agents to catch failures before users](#q11)
12. [Cost and latency at 10x usage](#q12)
13. [Diagnosing and correcting a wrong answer](#q13)
14. [Versioning and rolling back agent behavior](#q14)
15. [Pause and resume with the same state](#q15)

---

<a id="q1"></a>
## Q1. How do you handle a tool call that fails or returns malformed output?
*Source: interview posts*

**What the interviewer is testing:** Classifying failures and choosing the right response for each, instead of a blanket "retry".

**Strong answer:**
"I first classify **what** failed, because each class needs a different response:

| Failure | Example | Response |
|---|---|---|
| Transient infra | Timeout, 503, connection reset | Retry with exponential backoff + jitter (if idempotent) |
| Rate limit | 429 | Honor `Retry-After`, queue, or fall back |
| Bad arguments from the model | Invalid ID format, 400 | Return the error **to the model** as an observation so it corrects itself |
| Business/permission error | 403, 'order not cancellable' | Don't retry; return it to the model, which explains or chooses another path |
| Malformed tool *response* | Truncated JSON, schema drift in the API | Validate against an expected response model; if invalid → retry once, then mark the tool degraded |
| Persistent outage | Tool down for minutes | Circuit breaker opens → fallback tool, cached data, or graceful 'can't do that right now' |

Principles:
- **Tool outputs are validated too**, not just inputs. I wrap each tool's response in a Pydantic model, because upstream APIs change without telling you.
- **Errors are structured observations**: `{"error": "invalid_arguments", "field": "order_id", "hint": "format ORD-12345678"}`. Models are very good at fixing their own calls when told exactly what was wrong.
- **Bounded:** at most 2 model-driven corrections per tool per run, then escalate. Otherwise you have created a loop.
- **Never expose raw stack traces** to the model. They waste tokens and can leak internals.

In one support agent, about 6% of CRM calls failed with 400s due to date formats. Returning a precise hint let the model fix roughly 90% of them on the next step, with no human involvement."

**Follow-ups:**
- *Who retries, the model or the code?* → Code handles transient failures transparently. The model only sees errors it can act on.
- *What if the tool partially succeeded?* → Return what succeeded plus what failed. Never report full success.

**Red flags:**
- "Wrap it in try/except and retry 3 times" for every error type.
- Retrying non-idempotent writes blindly.

---

<a id="q2"></a>
## Q2. How do you validate structured output before acting on it?
*Source: interview posts*

**What the interviewer is testing:** Layered validation (syntax → schema → semantics → policy) and a repair loop.

**Strong answer:**
"Validation happens in layers, and the model's output is **untrusted input** until it passes all of them:

1. **Syntactic:** does it parse? Native tool calling or strict/constrained JSON modes make this nearly guaranteed.
2. **Schema:** Pydantic types, enums, ranges, required fields, no extras.
3. **Semantic/business:** does the referenced entity exist? Is the amount within the refund policy? Is the date in the future? Is it consistent with the retrieved data?
4. **Policy/authorization:** is this user allowed to do this, and does it need approval?

On failure I run a **bounded repair loop** that tells the model exactly what's wrong:

```python
from pydantic import BaseModel, Field, ValidationError, field_validator
from datetime import date

class LeaveRequest(BaseModel):
    employee_id: str
    start: date
    end: date
    leave_type: str = Field(pattern="^(casual|sick|earned)$")

    @field_validator("end")
    @classmethod
    def end_after_start(cls, v, info):
        if "start" in info.data and v < info.data["start"]:
            raise ValueError("end must be on/after start")
        return v

def get_validated(llm_call, prompt: str, max_repairs: int = 2) -> LeaveRequest:
    raw = llm_call(prompt)
    for attempt in range(max_repairs + 1):
        try:
            obj = LeaveRequest.model_validate_json(raw)
            check_business_rules(obj)          # balance, holidays, overlap -> raises BusinessError
            return obj
        except (ValidationError, BusinessError) as e:
            if attempt == max_repairs:
                raise
            raw = llm_call(f"{prompt}\n\nYour previous output:\n{raw}\n"
                           f"It failed validation:\n{e}\nReturn corrected JSON only.")
```

Notes:
- Repair prompts include the **specific** error. That fixes most cases in one attempt.
- Cap repairs at 2. After that it's a genuine failure: escalate or ask the user.
- For high-stakes actions, add a **confirmation step** showing the parsed values to the user ('Apply casual leave 3–5 Oct?').
- Log the validation failure rate per field. A spike usually means a prompt or model change regressed.

On an extraction pipeline, schema mode plus a single repair step took the valid rate from 96.1% to 99.7%."

**Follow-ups:**
- *Constrained decoding vs validate-and-repair?* → Constrained decoding guarantees syntax and schema but not semantics, so you still need layers 3–4.
- *What about hallucinated IDs?* → Check existence against the source of truth. Never trust an ID the model wasn't given.

**Red flags:**
- `json.loads` and go.
- Unbounded "try again" loops.

---

<a id="q3"></a>
## Q3. The agent is stuck calling the same tool. How do you stop it safely and prevent infinite loops and excessive tool calls?
*Source: interview posts*

**What the interviewer is testing:** Layered loop protection, graceful termination, and root-cause thinking.

**Strong answer:**
"Loops come from a few causes: the tool returns something the model misreads (empty results, a vague error), the goal is ambiguous, or there's no clear stop criterion. I defend at several layers.

**Hard guards (orchestrator, non-negotiable):**
- `max_steps` per run, `max_calls_per_tool`, a token/cost budget, and wall-clock time.
- **Repeat detection:** hash `(tool, normalized_args)`. On the 2nd identical call, return the cached result plus a hint. On the 3rd, block.
- **No-progress detection:** if the state (new facts, artifacts) hasn't changed for K steps, stop.
- **Oscillation detection:** A→B→A→B patterns in the recent window.

```python
import hashlib, json
from collections import Counter

class LoopGuard:
    def __init__(self, max_steps=12, max_same_call=2, max_tokens=80_000, max_per_tool=5):
        self.max_steps, self.max_same, self.max_tokens, self.max_per_tool = \
            max_steps, max_same_call, max_tokens, max_per_tool
        self.steps, self.tokens = 0, 0
        self.calls, self.per_tool = Counter(), Counter()

    def check(self, tool: str, args: dict, tokens: int) -> str | None:
        self.steps += 1; self.tokens += tokens
        key = hashlib.sha256(f"{tool}:{json.dumps(args, sort_keys=True)}".encode()).hexdigest()
        self.calls[key] += 1; self.per_tool[tool] += 1
        if self.steps > self.max_steps:           return "max_steps"
        if self.tokens > self.max_tokens:         return "token_budget"
        if self.calls[key] > self.max_same:       return "repeated_call"
        if self.per_tool[tool] > self.max_per_tool: return "tool_overuse"
        return None
```

**Soft steering (before the hard stop):** inject an observation such as 'You've called `search` 3 times with similar queries and no new results. Answer with what you have or ask the user.' That resolves most loops.

**Stopping safely:** return a partial answer plus what was tried, or escalate. Checkpoint the state so a human or a later run can resume. Never kill it silently mid-write.

**Root cause:** the loop rate is a metric. I sample the looping traces weekly. The fixes are usually better tool responses ('0 results for X; try broader terms' rather than `[]`), clearer descriptions, or a clarifying-question tool. In one project, changing empty-result responses alone cut loop terminations from 3.8% to 0.6%."

**Follow-ups:**
- *What limit values?* → From data: roughly the P99 step count of successful runs plus a margin. Tighter for customer-facing, looser for background research.
- *Temperature's role?* → At temp 0 the model repeats deterministically. Varying the hint breaks loops better than raising temperature.

**Red flags:**
- Only `max_iterations`, with no understanding of why loops happen.
- A hard stop that returns an exception to the user.

---

<a id="q4"></a>
## Q4. How do you handle tool timeouts, retries and rate limits?
*Source: interview posts*

**What the interviewer is testing:** Standard distributed-systems resilience applied to agents.

**Strong answer:**
"Tools are network calls, so the normal resilience toolkit applies, tuned for agents:

**Timeouts**
- A per-tool timeout based on its latency profile (P99 × 1.5). For example 3 s for a lookup, 30 s for a report.
- A **run-level deadline** that propagates down, so a tool never gets more time than the run has left.
- On timeout, return a structured error to the model. For slow tools, go async: `start_report` → job ID → `check_report`.

**Retries**
- Only for transient errors (timeouts, 5xx, connection errors), and only if the call is **idempotent** or carries an idempotency key.
- Exponential backoff with jitter: `base * 2^n + random(0, base)`, max 3 attempts.
- Retry budgets: cap total retries across the run so an outage doesn't multiply load (retry storms).

**Rate limits**
- Honor `Retry-After` on 429.
- A client-side token bucket per tool/provider, shared across workers (Redis), so we throttle ourselves before the provider does.
- Priority queues: interactive users go ahead of batch jobs.
- For LLM providers: track TPM/RPM and route to a secondary deployment or region when near the limit.

**Circuit breaker**
- If the error rate for a tool exceeds 50% over 30 s, open the circuit and fail fast for 60 s, then half-open to probe. Tell the model: 'CRM is temporarily unavailable'.

```python
import asyncio, random

async def call_with_retry(fn, *, timeout, attempts=3, base=0.5, retry_on=(TimeoutError, TransientError)):
    for n in range(attempts):
        try:
            return await asyncio.wait_for(fn(), timeout=timeout)
        except retry_on:
            if n == attempts - 1:
                raise
            await asyncio.sleep(base * 2 ** n + random.uniform(0, base))
```

Libraries like `tenacity` do this well. What matters is that the policy is **per tool**, because a payment call and a search call need different policies."

**Follow-ups:**
- *Retry at the tool level or the agent level?* → The tool level, for transient failures. The agent level (model re-plans) for semantic failures. Never both blindly, or retries multiply.
- *What does the user see during retries?* → Streaming status ('Checking your order…'). Past a latency SLO, degrade gracefully.

**Red flags:**
- Infinite or immediate retries with no jitter.
- The same timeout for every tool.

---

<a id="q5"></a>
## Q5. How do you design retries that don't cause duplicate side effects? How do you make tool execution idempotent?
*Source: interview posts*

**What the interviewer is testing:** Idempotency keys, deduplication, and understanding that at-least-once delivery is the reality.

**Strong answer:**
"The trap: the email API times out, but it *did* send. We retry and the customer gets two emails. Or worse, two refunds. Retries give you at-least-once execution. **Idempotency** turns that into effectively-once.

**1. Idempotency key per logical action**, derived deterministically, not randomly per attempt:
`key = hash(run_id, step_id, tool, canonical_args)`

The same run, step and arguments always produce the same key, across retries, crashes and resumes.

**2. Execute-once store:**

```python
import hashlib, json

def idem_key(run_id: str, step_id: str, tool: str, args: dict) -> str:
    return hashlib.sha256(f"{run_id}|{step_id}|{tool}|{json.dumps(args, sort_keys=True)}".encode()).hexdigest()

def execute_idempotent(db, run_id, step_id, tool, args, fn):
    key = idem_key(run_id, step_id, tool, args)
    # INSERT ... ON CONFLICT DO NOTHING -> atomic claim
    claimed = db.try_insert("tool_executions", key=key, status="IN_PROGRESS")
    if not claimed:
        row = db.get("tool_executions", key)
        if row.status == "DONE":
            return row.result                      # replay: return stored result, no re-execution
        raise InProgressError(key)                 # another worker has it; wait/poll
    try:
        result = fn(**args, idempotency_key=key)  # pass downstream too (Stripe-style)
        db.update("tool_executions", key, status="DONE", result=result)
        return result
    except Exception:
        db.update("tool_executions", key, status="FAILED")
        raise
```

**3. Pass the key downstream.** Many APIs (Stripe, many payment gateways, internal services) accept an `Idempotency-Key` header and deduplicate server-side. That covers the 'it succeeded but we timed out' case, which a local store alone can't.

**4. For APIs without idempotency support:** do a read before retrying ('did an email with message-ID X go out?'), use natural unique constraints (a unique `order_id + refund_reason`), or put the action behind an outbox/queue with exactly-once consumer semantics.

**5. Watch for the LLM itself.** After a crash-resume, the model may decide to 'send the email again'. The orchestrator must check the execution log before running any side-effecting tool, and tell the model 'already done at 10:42'.

Reads are naturally idempotent. Writes must be designed to be."

**Follow-ups:**
- *Why not a random UUID per call?* → Each retry would get a new key and deduplication would fail. The key must identify the *intent*, not the attempt.
- *How long do you keep keys?* → At least the retry and resume window. 24 h–7 days is typical.

**Red flags:**
- "We just don't retry writes," which leaves you with lost actions instead.
- Generating the idempotency key inside the retry loop.

---

<a id="q6"></a>
## Q6. How do you handle partial failures and recover without restarting the whole workflow?
*Source: interview posts*

**What the interviewer is testing:** Step-level durability, compensation, and degraded completion.

**Strong answer:**
"Design the workflow as **durable steps** with recorded outcomes, so recovery means 'resume from the failed step', not 'start over'.

**1. Persist after every step:** step ID, inputs, outputs, status. That's the checkpoint (Q8).

**2. Classify the failed step:**
- **Retryable** → retry with its policy.
- **Replaceable** → a fallback tool or source (backup search API, cached data).
- **Optional** → skip it and mark the result as partial. For example, the competitor-pricing step failed, but the report still ships with a note.
- **Critical** → pause and escalate to a human with full context, or run compensation.

**3. Compensation (saga pattern)** for multi-step side effects: each write step has an undo. If 'book hotel' succeeded and 'book flight' fails permanently, run `cancel_hotel`. In practice it's better to reorder steps so the hard-to-undo action goes last, and to use holds or reservations before commits.

**4. Isolate parallel branches.** In a fan-out of 5 research sub-tasks, one failure shouldn't cancel the other four. Gather with `return_exceptions=True`, then let the synthesizer work with 4/5 and state what's missing.

**5. Resume.** On restart the orchestrator loads the run state, skips `DONE` steps (their outputs are reused), and re-executes from the first non-done step with idempotency keys.

Example: in a 9-step onboarding agent, step 7 ('provision laptop request' in ServiceNow) failed during an outage. Steps 1–6 were checkpointed. The run paused with status `WAITING_RETRY`, retried automatically after 15 min, and completed. The user saw 'All set except the laptop request, which will be submitted shortly', not a failure."

**Follow-ups:**
- *What goes in the partial result?* → What completed, what failed, why, and what the user or system will do next.
- *When do you prefer compensation over holds?* → Holds or two-phase actions when the API supports them. Compensation otherwise.

**Red flags:**
- Restarting from scratch, which re-executes side effects.
- All-or-nothing results for independent branches.

---

<a id="q7"></a>
## Q7. Design retry, timeout, fallback and recovery mechanisms for agents.
*Source: interview posts*

**What the interviewer is testing:** Bringing it together into a coherent resilience architecture across every layer.

**Strong answer:**
"I design resilience per layer, with explicit policies:

```
Run level     : deadline (e.g. 60s interactive), budget, max_steps, checkpoint per step
  │
Model level   : timeout per call, retry on 5xx/429, fallback model/provider, output repair
  │
Tool level    : per-tool timeout, retry policy, idempotency, circuit breaker, fallback tool
  │
Data level    : cached/last-known-good, stale-while-revalidate
  │
Human level   : escalation queue with full trace when automation can't recover
```

**Policy table (declared as config, not scattered in code):**

| Component | Timeout | Retries | Fallback | On final failure |
|---|---|---|---|---|
| Primary LLM | 30 s | 2 (5xx/429) | Secondary provider/region, then smaller model | Graceful message |
| `get_order` | 3 s | 3, backoff | Read replica / cache (≤5 min stale) | Observation to model |
| `web_search` | 8 s | 1 | Alternate search API | Skip, mark partial |
| `issue_refund` | 10 s | 2 **with idempotency key** | None | Pause → human queue |

**Fallback hierarchy for the LLM:** the same model in another region → an equivalent model from another provider (prompts pre-tested on it) → a smaller model with a reduced feature set → a static response plus a ticket. Fallback models run the eval suite in CI too, so failover doesn't mean a quality cliff.

**Recovery:** durable execution (checkpoint per step, via LangGraph checkpointer, Temporal or a custom Postgres-backed state) means a worker crash only loses the in-flight step.

**Observability of resilience:** metrics for retries per tool, fallback activation rate, circuit states, and runs escalated. If the fallback rate exceeds 5%, page someone. Silent fallbacks can hide a degraded experience for weeks.

The principle: **fail fast, fail informatively, degrade gracefully, never duplicate side effects.**"

**Follow-ups:**
- *Would you use Temporal?* → For long-running, business-critical flows with many side effects, yes. It gives durable execution, retries and timers for free. For short chat agents, the LangGraph checkpointer or Postgres state is enough.
- *How do you test this?* → Chaos/fault injection in staging: mock tools that time out, 429, or return garbage, with assertions on outcomes.

**Red flags:**
- Resilience only at the HTTP client level.
- Fallback models that were never evaluated.

---

<a id="q8"></a>
## Q8. How do you handle long-running tasks (hours or days) with checkpointing, including recovery after a worker crashes halfway?
*Source: interview posts*

**What the interviewer is testing:** Durable execution design: state externalization, leases, resumability and idempotent replay.

**Strong answer:**
"Long-running agents must be **stateless workers over durable state**. The worker process is disposable. The run lives in a database.

**State model (per run):**
- run_id, status (`RUNNING | WAITING_HUMAN | WAITING_TIMER | DONE | FAILED`), version (prompt/workflow version pinned at start)
- the plan, current step, and the step log (inputs, outputs, status, idempotency key)
- messages or a compacted context (summaries of old steps plus references to artifacts)
- artifacts in object storage (reports, files), stored as references, never inline

**Checkpointing:** write after **every** step transition, in the same transaction as the step's recorded result. Checkpoint content includes enough to rebuild the prompt: the summary so far, the current plan, and the last N messages.

**Crash recovery:**
1. Workers take a **lease** on a run (`locked_by`, `lease_expires_at`) and heartbeat every 30 s.
2. If the worker dies, the lease expires and another worker claims the run.
3. It loads the last checkpoint. `DONE` steps are skipped. The in-flight step is re-executed, safely because of idempotency keys (the side effect happened at most once).

```python
def resume(run_id: str):
    run = store.claim(run_id, worker_id, lease_s=90)       # atomic, fails if leased
    for step in run.plan.steps:
        if run.step_status(step.id) == "DONE":
            continue                                        # reuse stored output
        out = execute_idempotent(db, run_id, step.id, step.tool, step.args, TOOLS[step.tool])
        store.checkpoint(run_id, step.id, out)              # atomic write + lease renew
        if step.requires_approval:
            store.set_status(run_id, "WAITING_HUMAN"); return  # worker exits; no resources held
    store.set_status(run_id, "DONE")
```

**Waiting for hours or days** (human approval, a report job, 'check again tomorrow'): don't hold a worker. Persist `WAITING_*` with a wake-up trigger (event, webhook or timer) and resume later.

**Context over days:** compact older steps into summaries, and keep references to raw outputs for retrieval on demand.

**Versioning:** a run started on workflow v12 finishes on v12, or is explicitly migrated. Don't hot-swap prompts mid-run.

Tooling: LangGraph with a Postgres checkpointer for agent graphs; Temporal or Step Functions when durability and timers are business-critical."

**Follow-ups:**
- *What if the crash happened after the side effect but before the checkpoint?* → The idempotency key plus the downstream dedup key returns the original result on replay. This is exactly why both exist.
- *How big do checkpoints get?* → Keep them small by storing large outputs as blob references and summarizing old messages.

**Red flags:**
- Keeping state in process memory.
- A long-running worker holding a connection open for days waiting for approval.

---

<a id="q9"></a>
## Q9. How would you design human-in-the-loop approval for high-risk actions?
*Source: interview posts*

**What the interviewer is testing:** Risk tiering, clean interrupt/resume mechanics, and avoiding rubber-stamp approvals.

**Strong answer:**
"Not every action needs approval, or approvals become noise and people rubber-stamp them. I **tier actions by risk**:

| Tier | Examples | Control |
|---|---|---|
| 0: Read | Lookups, search | Auto |
| 1: Low-risk reversible write | Draft email, create ticket | Auto + audit log + undo |
| 2: Moderate / external-facing | Send email to customer, refund ≤ ₹500 | Auto within policy limits; sample review |
| 3: High-risk / irreversible | Refund > ₹500, delete data, payments, prod changes, executing generated code on real systems | **Mandatory approval** |

Risk is decided by **code** (tool metadata plus argument thresholds plus user role), never by the model deciding whether it needs approval.

**Mechanics:**
1. The agent proposes the action. The orchestrator intercepts before execution (LangGraph `interrupt()` / `interrupt_before`, or a custom gate).
2. It checkpoints the state, sets `WAITING_APPROVAL`, and releases the worker.
3. It sends an approval card (Slack/UI) with the **exact action and arguments**, the agent's rationale, evidence and citations, impact ('refund ₹4,200 to card ending 1234'), and approve / edit / reject.
4. The approver's decision is recorded (who, when, what changed).
5. On resume: execute with an idempotency key. On edit: execute the edited arguments. On reject: the reason goes back to the agent as an observation.
6. **Timeout/expiry:** if not approved in N hours, escalate or cancel. Stale approvals are re-validated (did the price or state change?) before execution.

**Avoiding rubber-stamping:** show diffs rather than walls of text, track the approval rate. If it's 99.8%, consider lowering the tier with policy limits. If it's low, the agent needs work.

Example: in a finance ops agent, 92% of actions were tier 0–1 and auto. Tier-3 approvals averaged 4 min turnaround. Tracking edit-on-approve rates showed us which prompts needed fixing."

**Follow-ups:**
- *Approval for auto-executing code?* → Sandbox execution can be auto (no network, resource limits). Anything touching real systems or data needs approval (file 17).
- *Who can approve?* → RBAC: approvers need authority for that action and amount, plus a four-eyes rule for the highest tier.

**Red flags:**
- Letting the LLM decide whether approval is needed.
- An approval UI showing only the agent's summary, not the exact action.

---

<a id="q10"></a>
## Q10. How do you control cost and set hard token-spend limits when an agent can call tools repeatedly?
*Source: interview posts*

**What the interviewer is testing:** Budget enforcement plus the engineering levers that reduce cost per task.

**Strong answer:**
"Two parts: **hard ceilings** so a runaway can't happen, and **efficiency** so normal runs are cheap.

**Hard ceilings (enforced in the orchestrator):**
- Per run: max steps, max input+output tokens, max cost in currency (computed from the usage returned on each call × price table).
- Per user/tenant: daily and monthly quotas in the LLM gateway, with a 429 or degraded mode when exceeded.
- Per tool: max calls per run (a web search agent capped at 5 searches).
- Budget-aware behavior: at 80% of budget, inject 'wrap up with what you have'. At 100%, stop and return a partial result.

**Why agents get expensive:** every step re-sends the whole growing context. Step 10 of a run might send 25k input tokens. Cost grows roughly quadratically with steps.

**Efficiency levers:**
| Lever | Typical impact |
|---|---|
| Prompt caching of system prompt + tool defs | 50–90% cheaper on cached tokens, lower TTFT |
| Trim/summarize tool outputs before they enter context | 30–60% fewer input tokens |
| Model tiering: small model for routing/extraction, large for hard reasoning | 40–70% lower cost |
| Workflow instead of agent for known paths | Fewer calls |
| Parallel tool calls in one turn | Fewer round-trips |
| Context compaction of old steps | Flattens the quadratic growth |
| Caching tool results (same query in a session) | Fewer calls |

**Visibility:** cost per run, per feature and per tenant on dashboards, plus anomaly alerts (cost per run > 3× P95).

Example: a research agent averaged ₹18/run with a P99 of ₹140. Caching plus output trimming plus a haiku-class model for sub-steps brought the average to ₹6. A hard cap of ₹40/run removed the tail."

**Follow-ups:**
- *How do you enforce limits across parallel sub-agents?* → A shared budget object (Redis counter) decremented atomically by all branches.
- *How do you price-check before a call?* → Count tokens up front (tokenizer or count API) and refuse a call that would exceed the remaining budget.

**Red flags:**
- Only monitoring the monthly bill.
- No per-run cap: "we trust the max_steps".

---

<a id="q11"></a>
## Q11. How do you monitor an agent in production to catch failures before users do?
*Source: interview posts*

**What the interviewer is testing:** Agent-specific observability: traces, quality signals, and alerting on leading indicators.

**Strong answer:**
"Standard APM isn't enough. A run can return 200 OK and still be wrong. I monitor on three levels.

**1. Tracing (the foundation):** every run is a trace. Every LLM call and tool call is a span with inputs, outputs, tokens, latency, model/prompt version, and user/tenant. Tools: Langfuse, LangSmith, Arize Phoenix, or OpenTelemetry GenAI conventions into our existing stack.

**2. Metrics and leading indicators:**
| Category | Metrics |
|---|---|
| Health | Run success rate, tool error rate per tool, fallback activation, circuit states |
| Behavior | Steps per run (distribution), loop/guard terminations, re-plan rate, tool-selection mix drift |
| Cost/latency | Tokens and cost per run, P50/P95/P99 latency, TTFT |
| Quality | Online LLM-judge scores on sampled runs (faithfulness, task success), refusal rate, escalation rate |
| User | Thumbs down, rephrase-and-retry rate, abandonment, human-takeover rate |

**3. Detection before users complain:**
- **Synthetic canaries:** a golden set of 50 tasks runs every 15 min against prod. It alerts on failure or a score drop.
- **Anomaly alerts:** tool error rate > 5%, steps-per-run P95 up 30%, cost per run up 2×, judge score down more than 5 points day over day.
- **Change correlation:** annotate dashboards with deploys, prompt and model versions, since most regressions line up with a change.
- **Provider-side changes:** pin model versions and alert on provider deprecation notices.

**4. Close the loop:** failed or flagged traces go to a review queue. Labelled failures become regression tests.

Example: canaries caught a CRM API field rename 40 minutes after the vendor deployed it. The tool response validation started failing, and we rolled out a mapping fix before the morning traffic peak."

**Follow-ups:**
- *How much do you sample for LLM-judge scoring?* → 1–5% of traffic plus 100% of flagged runs, stratified by feature.
- *PII in traces?* → Redact at ingestion, apply role-based access to traces, and set retention limits.

**Red flags:**
- "We check the logs when users complain."
- Only infra metrics, with no quality signal.

---

<a id="q12"></a>
## Q12. What happens to cost and latency at 10x current usage?
*Source: interview posts*

**What the interviewer is testing:** Capacity thinking: what scales linearly, what breaks, and what to do about it.

**Strong answer:**
"I'd walk through what scales linearly and what breaks first.

**Cost:** roughly linear at 10x for LLM tokens, so the bill is 10x unless per-unit cost changes. At 10x, the levers are worth their engineering effort:
- prompt caching and semantic/result caching. Cache hit rates *improve* with volume because there are more repeated queries.
- model tiering, and moving known paths from agents to workflows
- volume discounts, committed or provisioned throughput, or self-hosting a smaller model for high-volume sub-tasks when the break-even works out (often when a single task type exceeds tens of millions of tokens a day)

**Latency: the non-linear part.**
- **Provider rate limits** (TPM/RPM) are usually the first wall. You get 429s, and queued requests blow up P95/P99. Fixes: raise quotas early, multi-region or multi-provider routing, priority queues.
- **Downstream tools** such as the CRM, SQL DB and vector DB were sized for 1x. The agent's fan-out multiplies load, since 1 user request might be 6 tool calls. You need connection pooling, read replicas, caching and circuit breakers.
- **Concurrency in our own service:** async workers, connection pools, autoscaling on queue depth rather than CPU.
- **Self-hosted models:** GPU capacity, and KV-cache pressure at high concurrency raises TTFT. Continuous batching and more replicas help.
- **Tail amplification:** with N sequential calls, run P99 grows fast. At 10x you see more tail events per run.

**How I'd prepare:**
1. A load test at 10x with realistic traffic mix (steps per run, tool mix).
2. Identify the first saturating component: usually rate limits, then a downstream API.
3. Capacity plan: tokens/min needed = users × runs/user/min × tokens/run. For example 5,000 concurrent users × 0.5 runs/min × 20k tokens ≈ 50M TPM, far above a default quota.
4. Degradation plan: shed low-priority traffic, shorter context, smaller model under load.

The key insight: agents multiply everything downstream, so at 10x the bottleneck is rarely the model itself."

**Follow-ups:**
- *Would you self-host at 10x?* → Only for stable, high-volume, well-evaluated sub-tasks where a smaller open model meets the quality bar and the GPU cost beats API cost at the real utilization.
- *How do you keep P95 in check?* → Parallelize independent calls, cap steps, stream partial results, hedged requests for tail-latency reads.

**Red flags:**
- "Just autoscale the pods."
- Ignoring provider rate limits and downstream systems.

---

<a id="q13"></a>
## Q13. An agent gave a wrong answer. How do you identify the cause and correct it?
*Source: interview posts*

**What the interviewer is testing:** A systematic, trace-driven debugging method, and turning the fix into a regression test.

**Strong answer:**
"I treat it like a production bug: reproduce, localize, fix, prevent.

**1. Get the trace.** Find the run by ID. With per-step tracing I can see exactly what the model saw and did.

**2. Localize by walking the chain.** Where did the truth get lost?

| Stage | Question | Typical fix |
|---|---|---|
| Understanding | Did it interpret the request correctly? | Prompt, clarifying-question tool |
| Planning/tool choice | Right tool? Right order? | Tool descriptions, routing |
| Arguments | Right args (dates, IDs, filters)? | Schema constraints, examples |
| Tool result | Did the tool return correct, complete data? | Tool/API bug, pagination, permissions |
| Context | Did the result actually reach the model (truncation, dropped, race condition)? | Plumbing, context budget |
| Reasoning/synthesis | Correct data but a wrong conclusion? | Prompt, stronger model, verifier step |
| Output | Right answer but mangled formatting or parsing? | Output parsing |

A common surprise is that it's not the model at all. Once, a tool response was truncated to 4 KB and the model never saw the relevant field. Another team had retrieval running fire-and-forget, so the prompt was built before the results arrived.

**3. Reproduce deterministically:** replay the run with recorded tool outputs (mocked from the trace) at temperature 0, then vary one factor at a time.

**4. Fix at the right layer.** Don't paper over a data bug with prompt text.

**5. Prevent:** add the case, and a few variants, to the regression eval set. Check whether it's systemic by querying traces for similar patterns (for example all runs where `get_order` returned a truncated response).

**6. Correct for the user:** if a wrong action was taken, compensate, notify, and log the incident.

**Correcting at runtime** (so fewer wrong answers ship): a verifier step for high-stakes answers that checks claims against tool outputs, and confidence-based escalation."

**Follow-ups:**
- *No tracing exists. What do you do first?* → Add it. Debugging agents without step-level traces is guesswork.
- *How do you know it's systemic vs a one-off?* → Query traces for the failure signature and check eval-set performance by slice.

**Red flags:**
- "Tweak the prompt until that example works."
- Not separating tool and data errors from reasoning errors.

---

<a id="q14"></a>
## Q14. How do you version and roll back an agent's behavior without redeploying the whole system?
*Source: interview posts*

**What the interviewer is testing:** Treating prompts, tools, models and workflows as versioned config, with safe rollout and instant rollback.

**Strong answer:**
"An agent's behavior is defined by several artifacts, and I version **all of them together** as an *agent version*:

```
agent_version: support-agent@v23
  system_prompt: prompts/support@sha:9f2c
  model: claude-sonnet-5 (pinned version), temperature 0.2
  tools: [get_order@v4, issue_refund@v2, kb_search@v7]
  workflow_graph: support_graph@v11
  guardrail_config: gr@v5
  limits: {max_steps: 12, budget_inr: 5}
  eval_baseline: evalrun-2026-09-20 (score 0.91)
```

**How:**
- **Stored as config** in a registry (Git-backed plus a DB, or a prompt-management tool like Langfuse, LangSmith or PromptLayer), loaded at runtime. The code is generic; behavior is data.
- **Immutable versions.** A change creates v24. Nothing is edited in place.
- **Routing by flag:** a feature-flag or config service maps traffic to a version (tenant, percentage, user segment). **Rollback = flip the pointer back to v23**, which takes seconds and needs no deploy.
- **Promotion gates:** v24 must pass the offline eval suite (no regressions beyond thresholds on any slice), then shadow traffic, then a canary at 5% with online metrics compared, then ramp up.
- **Runs pin their version at start**, so long-running runs don't change behavior mid-flight.
- **Every trace records the agent version**, which makes comparisons and incident analysis easy.
- **Tool compatibility:** tool contracts are versioned too. v23 and v24 may reference different tool versions, so old ones stay available until drained.

What still needs a deploy: new tool *code* or new graph node types. Keep those backward-compatible and roll them out dark (deployed but unreferenced) before a config version starts using them.

Example: a prompt change improved tone but raised the refund-tool call rate by 18% in canary. We flipped back in under a minute and fixed it offline."

**Follow-ups:**
- *Why pin model versions?* → Provider alias updates (like `-latest`) can change behavior silently. Upgrades should be deliberate, eval-gated versions.
- *How do you compare versions online?* → A/B or canary on task success, escalation rate, cost, latency and judge scores, with statistical significance checks.

**Red flags:**
- Prompts hard-coded in the source, so changes need a full release.
- Versioning prompts but not the model, tools and graph.

---

<a id="q15"></a>
## Q15. How do you pause an agent mid-execution and resume it later with the same state?
*Source: interview posts*

**What the interviewer is testing:** Interrupt mechanics, state serialization, and validity when resuming later.

**Strong answer:**
"Pausing is a special case of checkpointing. The run's state is persisted at a step boundary, and the worker exits.

**Why pause:** human approval, a missing user input, waiting on an async job, rate-limit backoff, operator intervention (a 'kill switch' or 'pause all refund agents'), or budget exhaustion pending approval.

**Mechanics:**
1. **Pause only at safe points:** step boundaries, never in the middle of a side-effecting call. External pause requests set a flag the orchestrator checks between steps.
2. **Persist the full resumable state:** messages or compacted context, plan and step pointer, pending tool call (if pausing *before* execution), agent version, budgets used, idempotency log.
3. **Status plus a resume trigger:** `PAUSED(reason=awaiting_approval, resume_on=event:approval_123)`.
4. **Resume:** load the checkpoint, **re-validate the world** (is the order still refundable? has the price changed? did the approval expire?), inject any new input (the human's decision or the user's answer) as the next message, and continue from the pointer.

In LangGraph this is built in: compile the graph with a checkpointer (Postgres/Redis), use `interrupt()` inside a node or `interrupt_before=["issue_refund"]`, and resume by invoking with the same `thread_id` and a `Command(resume=...)` value.

```python
from langgraph.types import interrupt, Command

def approval_node(state):
    decision = interrupt({"action": state["pending_action"]})   # pauses, state saved
    return {"approved": decision == "approve"}

graph = builder.compile(checkpointer=postgres_saver)
cfg = {"configurable": {"thread_id": "run-8841"}}
graph.invoke(inputs, cfg)                      # runs until interrupt
# ... hours later, from the approval webhook:
graph.invoke(Command(resume="approve"), cfg)   # continues with identical state
```

**Pitfalls:** state that includes non-serializable objects (clients, file handles), because only data belongs in state. Schema changes between pause and resume need a state version plus migration. Context that's stale after days should be refreshed on resume."

**Follow-ups:**
- *Can you pause mid-LLM call?* → You'd cancel it and redo that step on resume. It has no side effects, so that's safe.
- *How do you pause everything in an incident?* → A global or per-agent kill-switch flag checked at every step boundary. Runs move to `PAUSED` cleanly.

**Red flags:**
- "Keep the process sleeping until approval."
- Resuming without re-checking that the world hasn't changed.

---

## Rapid-fire recap
- Classify failures: transient → retry; bad args → tell the model; business/permission → don't retry; outage → circuit breaker + fallback.
- Validate in layers: syntax → schema → business rules → policy. Bounded repair loop with specific errors.
- Loop guards: max steps, token/cost budget, repeat-call hash, no-progress detection. Steer softly before the hard stop.
- Retries only on idempotent calls, with exponential backoff + jitter and a retry budget.
- Idempotency key = hash(run, step, tool, args), stored atomically and passed downstream.
- Durable steps + checkpoints mean you resume from the failed step, never restart side effects.
- Long-running = stateless workers + durable state + leases + heartbeats. Wait without holding workers.
- HITL by risk tier decided by code. Show the exact action, re-validate on resume, expire stale approvals.
- Agent cost grows with steps × context. Cache, trim, tier models, compact, and hard-cap per run and tenant.
- Monitor traces, behavior metrics, online judge scores and synthetic canaries. Annotate deploys.
- At 10x, rate limits and downstream tools break before the model does.
- Version prompt + model + tools + graph together. Rollback = flip a pointer; runs pin their version.
