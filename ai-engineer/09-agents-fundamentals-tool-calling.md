# 09 — Agents: Fundamentals & Tool Calling

**What interviewers probe here:** whether you understand what is happening underneath the framework: how a model "calls" a tool, who actually executes it, what stops the loop, and when an agent is the wrong tool entirely. Nearly every AI engineering round now has an agent section, and these questions repeat across companies.

[← Back to index](README.md)

1. [What makes a system truly agentic? Chatbot vs agent vs workflow](#q1)
2. [Agentic architecture walk-through](#q2)
3. [How function/tool calling works under the hood](#q3)
4. [How the LLM decides which tool to call, and why descriptions matter](#q4)
5. [Defining, registering and executing tools](#q5)
6. [Designing a safe tool schema](#q6)
7. [What decides when an agent stops](#q7)
8. [Single-step vs multi-step planning, and task decomposition](#q8)
9. [ReAct, and ReAct vs Plan-and-Execute](#q9)
10. [When the plan must change mid-execution](#q10)
11. [Write the ReAct loop yourself](#q11)
12. [When NOT to use an AI agent](#q12)
13. [Routing between SQL, vector DB, API and web](#q13)
14. [Valid JSON, wrong tool](#q14)
15. [Walk-through: 3-day trip within ₹10,000](#q15)
16. [What makes a production agent more than an LLM](#q16)

---

<a id="q1"></a>
## Q1. What makes a system truly agentic? Chatbot vs agent vs workflow.
*Source: interview posts*

**What the interviewer is testing:** Whether you can draw the line by *who controls the control flow*, not by buzzwords.

**Strong answer:**
"The cleanest way to separate them is to ask who decides the next step.

| | Who decides the next step | Takes actions? | Example |
|---|---|---|---|
| Chatbot | Nobody; it's one request, one response | No | FAQ bot answering from its prompt |
| Workflow (LLM pipeline) | **The developer's code**; the path is fixed in advance | Maybe, but on fixed steps | Classify ticket → retrieve → draft reply |
| Agent | **The model**, at runtime, in a loop | Yes, via tools | "Resolve this refund": it looks up the order, checks policy, issues the refund |

A system is agentic when three things are true: it runs a **loop** (observe → decide → act → observe), the **LLM picks the next action** from a set of tools, and it **uses feedback** from tool results to adapt until a goal is met or a stop condition fires.

Agency is a spectrum, not a switch. A router that picks one of three chains has a little autonomy. A ReAct agent picking among 20 tools has a lot. In production I push *down* the spectrum whenever I can, because every bit of autonomy costs predictability, latency and testability. At my last project we started with a fully autonomous support agent. We ended up with a workflow in which only two steps were agentic: diagnosis and choosing a resolution. That cut P95 latency from 14 s to 5 s and made evals far more stable."

**Follow-ups:**
- *Is RAG an agent?* → Plain RAG is a workflow. It becomes agentic when the model decides whether, what and how many times to retrieve.
- *Is "agentic AI" different from "an agent"?* → People usually mean the whole system: agents plus orchestration, memory, guardrails and HITL, pursuing multi-step goals.

**Red flags:**
- "An agent is an LLM with a big prompt."
- Calling any LangChain chain an "agent".
- Saying you'd always use agents because they're more powerful.

---

<a id="q2"></a>
## Q2. Walk through agentic architecture: agent core, planning, memory, tools, execution, observation, re-planning.
*Source: interview posts*

**What the interviewer is testing:** Whether you see the LLM as one component inside a larger system, and know what each surrounding component is responsible for.

**Strong answer:**
"The LLM supplies the reasoning. The architecture around it decides what the system can actually do, and how safely.

```
User goal
   │
   ▼
┌──────────────────────── Orchestrator (code) ────────────────────────┐
│  State: messages, plan, step count, budget, artifacts               │
│                                                                     │
│  ┌──────────┐   decide    ┌───────────┐  validated   ┌───────────┐  │
│  │ LLM core │ ──────────► │ Tool call │ ───────────► │ Executor  │  │
│  │(reason + │             │  (JSON)   │  + authZ     │ (runtime) │  │
│  │  plan)   │ ◄────────── │           │ ◄─────────── │           │  │
│  └──────────┘ observation └───────────┘    result    └───────────┘  │
│       ▲                                                   │         │
│       │ retrieve                                          ▼         │
│  ┌──────────┐                                   APIs / DB / search  │
│  │  Memory  │  short-term = message state; long-term = store        │
│  └──────────┘                                                       │
│  Guardrails · Budgets · Tracing · HITL gates                        │
└─────────────────────────────────────────────────────────────────────┘
```

- **Agent core (LLM):** interprets the goal and chooses the next action or final answer.
- **Planning:** implicit (ReAct, one step at a time) or explicit (write a plan, then execute it).
- **Memory:** short-term working state (the conversation plus scratchpad) and long-term stores (user preferences, past episodes, a knowledge base).
- **Tools:** typed capabilities such as search, SQL, APIs or code.
- **Execution:** always done by *my code*, never by the model. This is where auth, validation, timeouts and idempotency live.
- **Observation and feedback:** tool results go back into context. Errors are returned as structured observations, not raised as exceptions.
- **Re-plan:** if the observation invalidates the plan, the model revises it.

The component people forget is the **orchestrator itself**. It enforces step limits, token budgets, approval gates and tracing. Most production incidents I've seen came from that layer, not from the model."

**Follow-ups:**
- *Where do guardrails sit?* → At input, before tool execution (policy/authZ), and at output. They are code-level checks, not just prompt text.
- *Where is state stored?* → In a durable store (Postgres/Redis) keyed by run ID, so runs survive crashes.

**Red flags:**
- Leaving out the orchestrator, or saying "the LLM executes the tool".
- Treating memory as "just the chat history".

---

<a id="q3"></a>
## Q3. How does function/tool calling actually work under the hood between the LLM and your runtime?
*Source: interview posts*

**What the interviewer is testing:** Whether you know the model only *emits text shaped like a call*, and your runtime does everything else.

**Strong answer:**
"The model never executes anything. The mechanics are:

1. **Declare.** I send tool definitions (name, description, JSON Schema for parameters) with the request. The provider serializes them into the prompt in a special format the model was fine-tuned on.
2. **Decide.** The model was trained, through SFT and RL on tool-use traces, to emit a structured block such as `tool_use {name: "get_order", input: {"order_id": "A123"}}` when a tool is useful. Many providers apply constrained decoding so the arguments parse against the schema. The response ends with a stop reason like `tool_use`.
3. **Execute.** My runtime parses the block, validates the arguments, checks permissions, runs the real function, and captures the result or error.
4. **Return.** I append the assistant's tool-call message and a `tool_result` message (matched by `tool_use_id`) to the conversation, then call the model again.
5. **Loop** until the model returns a normal text answer (`stop_reason = end_turn`) or my limits trigger.

```
Request: messages + tools[] ──► Model ──► {tool_use: get_order(A123)}  stop_reason=tool_use
Runtime: validate → authZ → get_order("A123") → {"status":"shipped"}
Request: messages + tool_use + tool_result ──► Model ──► "Your order shipped yesterday."
```

Practical consequences:
- The model is **stateless**. Every turn re-sends the full history plus tool definitions, so 30 verbose tools can add 3–5k tokens to *every* call. Prompt caching helps here.
- Models can emit **parallel tool calls** in one turn. My executor must handle several and return every result.
- Arguments can still be semantically wrong even when they are schema-valid, so validation is my job."

**Follow-ups:**
- *What's `tool_choice`?* → It controls whether the model may, must, or must call a specific tool (`auto` / `any` / `{name}` / `none`).
- *How did models do this before native support?* → Prompt the model to emit JSON, then parse it with a regex. That was fragile, which is why native tool calling and constrained decoding won out.

**Red flags:**
- "The LLM calls the API."
- Not knowing that tool results must be sent back to the model for it to use them.

---

<a id="q4"></a>
## Q4. How does the LLM decide which tool to call, and why do tool descriptions matter so much?
*Source: interview posts*

**What the interviewer is testing:** Whether you understand that tool selection is next-token prediction conditioned on names, descriptions and schema, and that you can engineer it.

**Strong answer:**
"There's no separate router inside the model. Tool selection is next-token prediction conditioned on the user request, the conversation so far, and the tool definitions in context. The name, description and parameter docs are effectively the prompt for that decision. So tool descriptions are prompt engineering with the highest leverage.

What I put in a good description:
- **What it does and when to use it**: 'Look up a single order by ID. Use when the user references a specific order.'
- **When *not* to use it**: 'Do not use for listing orders; use `search_orders`.'
- **What it returns**, so the model can plan its next step.
- **Parameter semantics, formats and enums**: `status: enum[open, closed]`, `date: YYYY-MM-DD`.

Other design levers:
- **Fewer, well-separated tools.** Selection accuracy drops as tools multiply and overlap. With 40+ tools I route first, either to a toolset by domain or by retrieving the top-k tools by embedding similarity.
- **Distinct names.** `search_docs` next to `search_documents` invites mistakes.
- **Examples** in the system prompt for ambiguous cases.

In one project, merely rewriting descriptions to add 'use when / don't use when' took tool-selection accuracy on our eval set from 78% to 93%, with no model change. I measure this with a labelled set of (query → expected tool, expected args) and track it per release."

**Follow-ups:**
- *How do you evaluate tool selection?* → A golden dataset of queries with expected tool calls; score tool-name accuracy, argument accuracy, and unnecessary-call rate.
- *Would you fine-tune for tool selection?* → Only after description fixes and routing plateau, and when the tool set is stable.

**Red flags:**
- One-word descriptions, or "the model just knows".
- Registering 60 tools in one agent and blaming the model for picking the wrong one.

---

<a id="q5"></a>
## Q5. How do you define, register and execute tools? Tools vs functions, schemas and arguments.
*Source: interview posts*

**What the interviewer is testing:** Whether you can implement a clean tool layer yourself, separating the model-facing contract from execution.

**Strong answer:**
"A **function** is plain code. A **tool** is that function plus a model-facing contract: name, description and input schema, along with runtime policy such as timeout, permissions and whether it has side effects. Frameworks like LangChain's `@tool` just generate that contract from type hints and the docstring.

My pattern: Pydantic defines the arguments. The JSON Schema is generated from the Pydantic model, and the same model validates the arguments at execution time, so there is a single source of truth.

```python
from pydantic import BaseModel, Field
from typing import Callable

class GetOrderArgs(BaseModel):
    order_id: str = Field(pattern=r"^[A-Z]\d{3,10}$", description="Order ID like A1234")

class Tool:
    def __init__(self, name: str, desc: str, args: type[BaseModel], fn: Callable,
                 side_effect: bool = False, timeout_s: float = 10):
        self.name, self.desc, self.args, self.fn = name, desc, args, fn
        self.side_effect, self.timeout_s = side_effect, timeout_s

    def spec(self) -> dict:  # what the model sees
        return {"name": self.name, "description": self.desc,
                "input_schema": self.args.model_json_schema()}

REGISTRY: dict[str, Tool] = {}
def register(t: Tool): REGISTRY[t.name] = t

def execute(name: str, raw_args: dict, user) -> dict:
    tool = REGISTRY.get(name)
    if not tool:
        return {"error": f"unknown tool {name}", "available": list(REGISTRY)}
    try:
        args = tool.args.model_validate(raw_args)
    except Exception as e:
        return {"error": "invalid_arguments", "details": str(e)}  # model can self-correct
    if not user.can_use(name):
        return {"error": "forbidden"}
    return {"ok": True, "data": tool.fn(**args.model_dump())}
```

Key choices: errors come back as **data** so the model can recover; authorization is checked **at execution time** against the real user, not the agent; results are **trimmed or summarized** before they re-enter context, because a raw 50 KB API response wastes tokens and buries the signal."

**Follow-ups:**
- *How do you handle tool results that are huge?* → Paginate, project to the needed fields, or store the full output as an artifact and return a handle plus summary.
- *Sync or async execution?* → Async, so parallel tool calls can run concurrently under per-tool timeouts.

**Red flags:**
- Hand-writing JSON schemas that drift away from the code.
- Letting exceptions crash the loop instead of returning them as observations.

---

<a id="q6"></a>
## Q6. How would you design a safe tool schema?
*Source: interview posts*

**What the interviewer is testing:** Least-privilege thinking, and constraining the action space at the schema level.

**Strong answer:**
"My principle is to make dangerous calls unrepresentable, then enforce everything else in code.

1. **Narrow, intent-level tools**, not generic power tools. Expose `refund_order(order_id, reason)` with a server-side cap, never `run_sql(query)` or `http_request(url, method, body)`.
2. **Tight types.** Use enums instead of free strings, regex patterns for IDs, min/max on numbers, `additionalProperties: false`, and required fields.
3. **No identity or authority in the arguments.** The model never passes `user_id` or `role`. They come from the authenticated session. Otherwise an injected prompt can say 'set user_id=admin'.
4. **Separate read and write tools**, and mark side effects in metadata (`side_effect=True`, `reversible=False`) so the orchestrator can apply approval gates.
5. **Server-side limits** the model can't override: `amount <= 500 INR` in the schema *and* in the handler, rate limits per run, and scoping to a tenant.
6. **Dry-run / two-phase actions** for risky operations: `prepare_transfer` returns a preview plus a token, and `commit_transfer(token)` requires approval.
7. **Idempotency key** on every write (see file 10).

```json
{
  "name": "issue_refund",
  "description": "Refund a delivered order. Max 500 INR. Requires order to be in DELIVERED state.",
  "input_schema": {
    "type": "object",
    "properties": {
      "order_id": {"type": "string", "pattern": "^ORD-[0-9]{8}$"},
      "amount_inr": {"type": "number", "minimum": 1, "maximum": 500},
      "reason": {"type": "string", "enum": ["damaged", "late", "wrong_item"]}
    },
    "required": ["order_id", "amount_inr", "reason"],
    "additionalProperties": false
  }
}
```

The schema is the first line of defense. The handler still re-checks everything, because schema adherence is probabilistic unless you use strict or constrained modes."

**Follow-ups:**
- *What if the business needs flexible queries?* → Offer parameterized query tools over a semantic layer, or read-only SQL on a restricted replica with row-level security.
- *How do you handle tool output safety?* → Treat it as untrusted input. Mark it as data and never follow instructions inside it (file 17).

**Red flags:**
- A `execute_python` or `run_shell` tool with no sandbox.
- Passing user identity or permissions as model-supplied arguments.

---

<a id="q7"></a>
## Q7. What decides when an agent stops and returns a final answer?
*Source: interview posts*

**What the interviewer is testing:** Whether you know that the model proposes stopping but the orchestrator enforces it.

**Strong answer:**
"There are two layers.

**1. The model's decision (the natural stop).** In native tool calling, the agent stops when the model returns a turn with no tool calls: `stop_reason = end_turn`, or in ReAct text, a `Final Answer:`. The model does this when it judges the goal is met, based on the system prompt ('answer once you have X'), the tool results, and its training.

**2. Hard stops enforced by the orchestrator, which the model can't talk its way past:**
- `max_steps` (for example 8–15 for a support agent)
- a token or cost budget per run
- a wall-clock timeout
- repeated-action detection (the same tool with the same arguments N times)
- no-progress detection (the state hasn't changed for K steps)
- an explicit terminal tool such as `submit_answer(answer, citations)` or `escalate_to_human()`

I like the **terminal tool** pattern for structured tasks. The run ends only when the model calls `submit_answer`, whose schema forces the answer, citations and confidence. That removes ambiguity about 'is this the final answer or thinking out loud'.

On a hard stop, the agent should fail **gracefully**: return a partial result plus what it tried, or escalate. It should never go silent. On a support agent we saw about 2% of runs hit `max_steps=12`. Tracing showed that most were ambiguous user requests, so we added an `ask_clarifying_question` tool, which dropped hard-stops below 0.5%."

**Follow-ups:**
- *How do you pick max_steps?* → Plot the step-count distribution of successful runs and set it at roughly P99 + margin. Alert on the rate of runs that hit the limit.
- *What if it stops too early?* → Add a verifier or checklist step. Requiring specific fields in the terminal tool also forces completeness.

**Red flags:**
- "It stops when the LLM is done", with no hard limits.
- A limit that throws an exception and returns nothing to the user.

---

<a id="q8"></a>
## Q8. Single-step vs multi-step planning agents: how does an agent decompose a complex task?
*Source: interview posts*

**What the interviewer is testing:** Understanding of planning strategies and their cost and reliability trade-offs.

**Strong answer:**
"A **single-step agent** makes one decision, usually one tool call, then answers. An example is 'what's the weather in Pune' → `get_weather` → answer. It's cheap, predictable and easy to eval.

A **multi-step planning agent** handles goals that need several dependent actions: 'Compare our Q3 churn with Q2 and email the summary to finance.' Decomposition strategies:

| Strategy | How it works | Good for |
|---|---|---|
| Implicit (ReAct) | Decide one next step at a time from observations | Exploratory tasks, unknown paths |
| Explicit plan-then-execute | LLM writes an ordered plan; an executor runs the steps | Known task shapes, auditability |
| Hierarchical | Goal → subgoals → steps, possibly delegated to sub-agents | Large tasks, parallelism |
| Plan as DAG | Steps with dependencies; independent steps run in parallel | Research, multi-source fetches |

Decomposition quality improves with:
- a **structured plan output** (a list of steps with tool, inputs, dependencies and success criteria), validated with Pydantic
- **domain templates** or few-shot examples of good plans
- limiting depth (for example at most 7 steps) so plans stay executable
- a **checkpoint after each step**: did it achieve its success criterion?

The churn example becomes: (1) query Q3 churn, (2) query Q2 churn, where 1 and 2 run in parallel, (3) compute the delta, (4) draft the summary, (5) *approval gate*, (6) send the email.

Trade-off: explicit plans are cheaper per step, because the executor can use a smaller model, and they're auditable. But they go stale when reality differs, which is why I add re-planning triggers (Q10)."

**Follow-ups:**
- *Which model plans vs executes?* → A stronger model plans and cheaper models execute well-defined steps, which often cuts cost 40–60%.
- *How do you evaluate a plan?* → Compare against reference plans for coverage and ordering, plus executability (does every step map to an available tool).

**Red flags:**
- Assuming every task needs multi-step planning.
- Plans as free text that can't be validated or executed.

---

<a id="q9"></a>
## Q9. What is the ReAct pattern, and why interleave reasoning and actions? ReAct vs Plan-and-Execute.
*Source: interview posts*

**What the interviewer is testing:** Whether you know why ReAct works and when an upfront plan is the better choice.

**Strong answer:**
"ReAct (Reason + Act, Yao et al. 2022) interleaves **Thought → Action → Observation**, repeated until a final answer:

```
Thought: I need the user's plan tier before answering about limits.
Action: get_account(user_id)
Observation: {"tier": "Pro", "seats": 40}
Thought: Pro allows 50 seats, so they have 10 left.
Final Answer: You can add 10 more seats on Pro.
```

**Why interleave instead of planning everything upfront?** Because each observation carries information the model didn't have at planning time. Reasoning grounded in real tool outputs reduces hallucination compared with pure chain-of-thought, and the agent can react to errors, empty results or surprises. With modern native tool calling, the 'Thought' is the model's reasoning or text, and the 'Action' is a structured tool call.

| | ReAct | Plan-and-Execute |
|---|---|---|
| Adaptivity | High; re-decides every step | Low, unless you add re-planning |
| LLM calls | One strong-model call per step | One planner call plus cheap executor calls |
| Cost/latency | Higher; full context re-sent every step | Lower; steps can run in parallel |
| Predictability/audit | Harder; the path emerges | Easier; the plan is inspectable and approvable |
| Failure mode | Loops, drifting off-task | Stale plan when an assumption breaks |
| Best for | Exploratory, short, uncertain tasks | Long, well-structured, multi-part tasks |

My default in production is a **hybrid**: plan upfront, execute steps (some with small ReAct loops inside), and re-plan only when a step fails or returns something unexpected. For a research agent that pulled from 6 sources, moving from pure ReAct to plan-and-execute with parallel fetches cut latency from about 40 s to 15 s and cost by about 45%, with equal eval scores."

**Follow-ups:**
- *Does ReAct need visible "Thought" text?* → No. With reasoning models or native tool calls, the reasoning is internal. The loop structure is what matters.
- *What's Reflexion?* → Adding a self-critique or memory of past failures between attempts to improve on retries.

**Red flags:**
- Only being able to recite "Thought/Action/Observation" without any trade-offs.
- Claiming one pattern is always better.

---

<a id="q10"></a>
## Q10. How do you handle a plan that must change mid-execution based on a tool result?
*Source: interview posts*

**What the interviewer is testing:** Designing re-planning triggers without thrashing, and preserving the work already completed.

**Strong answer:**
"I treat the plan as **mutable state** with explicit triggers for changing it, not something the model rewrites on a whim.

**Re-plan triggers:**
- A step **fails** after retries (tool down, permission denied).
- A step's **success criterion** isn't met. For example, the search returned 0 results or the data is outside the expected range.
- An observation **invalidates an assumption** in a later step. For example, the flight is sold out, or the user is on the Free tier and the plan assumed Pro.
- The **user interjects** with new constraints.

**Mechanics:**
1. After each step, a cheap check (code first, LLM if needed) compares the result to the step's expected outcome.
2. On a trigger, call the planner with: the original goal, **completed steps and their results** (which are kept, never redone), the failure or observation, and the remaining steps.
3. The planner returns a revised remainder of the plan. I diff it and log the reason for the change.
4. **Guards against thrash:** at most 2–3 re-plans per run, and never re-plan into a plan identical to one that already failed. If the limit is exceeded, escalate with a partial result.
5. Side-effecting steps already done are **never re-executed**; the idempotency keys guarantee it.

Example: in a travel agent, step 3 'book hotel X' returns 'sold out'. The re-planner keeps the flight already found in step 2, swaps in 'search alternatives within 2 km under ₹3,000', and continues. It does not start over.

Trade-off: re-planning costs an extra strong-model call, so I only invoke it on triggers, not after every step."

**Follow-ups:**
- *What if the change needs user input?* → Pause (a HITL interrupt), checkpoint the state, ask a specific question, and resume.
- *How do you test re-planning?* → Inject failures in eval, using mocked tools that return sold-out, empty or error responses, and assert the agent recovers.

**Red flags:**
- "Just restart the agent."
- Re-planning that repeats completed side-effecting actions.

---

<a id="q11"></a>
## Q11. Write the ReAct loop yourself, with no LangChain.
*Source: interview posts*

**What the interviewer is testing:** Whether you can build the core loop from scratch, with the production details included.

**Strong answer:**
"Here's a native tool-calling loop in the Anthropic Messages style. The same shape works with any provider. It includes the parts people skip: step limits, repeated-call detection, errors returned as observations, and parallel tool calls.

```python
import json, anthropic

client = anthropic.Anthropic()
MODEL = "claude-sonnet-5"
TOOLS = [t.spec() for t in REGISTRY.values()]          # from Q5
SYSTEM = ("You are a support agent. Use tools to get facts; never guess. "
          "When you have enough information, answer concisely with sources.")

def run_agent(user_msg: str, user, max_steps: int = 10, max_tokens_total: int = 60_000):
    messages = [{"role": "user", "content": user_msg}]
    seen_calls: dict[str, int] = {}
    tokens_used = 0

    for step in range(max_steps):
        resp = client.messages.create(model=MODEL, system=SYSTEM, tools=TOOLS,
                                      messages=messages, max_tokens=1024)
        tokens_used += resp.usage.input_tokens + resp.usage.output_tokens
        messages.append({"role": "assistant", "content": resp.content})

        if resp.stop_reason != "tool_use":                # model chose to answer
            return "".join(b.text for b in resp.content if b.type == "text")

        results = []
        for block in resp.content:
            if block.type != "tool_use":
                continue
            key = f"{block.name}:{json.dumps(block.input, sort_keys=True)}"
            seen_calls[key] = seen_calls.get(key, 0) + 1
            if seen_calls[key] > 2:
                out = {"error": "repeated_call", "hint": "You already tried this. Change approach or answer."}
            else:
                out = execute(block.name, block.input, user)   # validation + authZ inside
            results.append({"type": "tool_result", "tool_use_id": block.id,
                            "content": json.dumps(out)[:8000],   # cap observation size
                            "is_error": "error" in out})
        messages.append({"role": "user", "content": results})

        if tokens_used > max_tokens_total:
            break

    return "I couldn't complete this reliably. Escalating to a human agent."  # graceful stop
```

What I'd point out while writing it:
- The **model decides** to stop (`stop_reason`); the **loop enforces** step and token limits.
- Every `tool_use` gets a matching `tool_result` by ID, or the API rejects the next call.
- Errors are **observations**, which lets the model self-correct (for example fixing a malformed ID).
- Observations are **capped** to protect the context window.
- In production I'd add tracing spans per step, per-tool timeouts with `asyncio`, prompt caching on the system prompt and tools, and a checkpoint after each step (file 10)."

**Follow-ups:**
- *Text-only ReAct for models without tool calling?* → Prompt for a `Action: name[json]` format, stop generation on `Observation:`, regex-parse, validate, inject the result. It's more fragile, so add a repair retry.
- *How do you run parallel tool calls?* → `asyncio.gather` over the tool_use blocks, each with its own timeout, then return every result in one user message.

**Red flags:**
- A `while True` loop with no limits.
- Letting a tool exception kill the run.

---

<a id="q12"></a>
## Q12. When should you NOT use an AI agent?
*Source: interview posts*

**What the interviewer is testing:** Senior judgment, and knowing when deterministic software is the better tool.

**Strong answer:**
"I'd skip an agent, and often skip the LLM entirely, when:

1. **The path is known.** If I can draw the flowchart, I write a workflow. Agents add non-determinism where none is needed.
2. **Correctness must be exact and auditable**: payroll calculation, tax, pricing, access control. Use rules or code, and let an LLM at most explain the result.
3. **Latency or cost budgets are tight.** An agent is N sequential LLM calls. A 300 ms autocomplete or a high-QPS classifier shouldn't be one.
4. **A simpler ML or rules solution works.** A fine-tuned classifier or regex that hits 98% beats an agent at 93% that costs 50x more.
5. **Actions are high-risk and irreversible** with no room for human review, for example moving money or deleting data.
6. **You can't evaluate it.** If there's no way to measure success, you can't safely ship autonomy.
7. **The data or tools aren't ready.** An agent over unreliable APIs just amplifies the flakiness.

My decision ladder: **code/rules → single LLM call → LLM workflow (fixed steps) → agent with a few tools → multi-agent.** Move up a rung only when the rung below demonstrably fails.

Example: a team wanted an 'invoice processing agent'. We shipped a workflow instead: extraction with structured output, deterministic validation rules, and the LLM only for exception explanations. It hit 97% straight-through processing, P95 of 3 s, and about ₹0.4 per invoice. The agent prototype was at 89% and five times the cost."

**Follow-ups:**
- *So where do agents shine?* → Open-ended tasks with variable paths and forgiving or reviewable outputs: research, triage, coding assistants, investigation.
- *How do you convince a stakeholder who wants 'an agent'?* → Frame it around outcomes (accuracy, cost, latency) and show a workflow baseline first.

**Red flags:**
- "Agents can do everything."
- No mention of determinism, cost or evaluability.

---

<a id="q13"></a>
## Q13. Design an agent that decides whether to query SQL, a vector DB, an API or the web.
*Source: interview posts*

**What the interviewer is testing:** Routing design, i.e. when to use a deterministic router vs LLM tool choice, and how to combine sources.

**Strong answer:**
"First I'd clarify what each source is authoritative for:

| Source | Authoritative for | Example query |
|---|---|---|
| SQL warehouse | Aggregates, exact numbers, structured facts | 'Revenue by region last quarter' |
| Vector DB (docs) | Policies, procedures, unstructured knowledge | 'What's our refund policy for enterprise?' |
| Internal API | Live, per-entity state | 'Status of order ORD-123' |
| Web search | Public, fresh, external info | 'Competitor's latest pricing' |

**Design:** a two-stage router.

```
Query ─► Router (small classifier / structured-output LLM call)
          ├─ intent: {sql, docs, api, web, multi, clarify}
          ├─ entities: order_id, dates, metric
          └─ confidence
      ├─ high confidence, single source ─► deterministic path (fast, cheap)
      └─ low confidence or multi-source ─► agent with all four tools (ReAct)
                                             └─► answer composer with citations
```

- **Stage 1** is cheap and deterministic-ish: a fine-tuned small classifier or a structured-output call on a small model with enums. It handles about 80% of traffic in ~100 ms.
- **Stage 2** is the full agent, for ambiguous or compound questions like 'Why did revenue drop in the South, and does it relate to the new policy?', which needs SQL plus docs.
- **Priority rules** in code: if an entity ID is present, prefer the API. Never use the web for internal data. Web is behind a feature flag and a domain allowlist.
- **Guardrails:** SQL goes through a read-only role, a semantic layer and a row limit. Web results are treated as untrusted content.
- **Evaluation:** a labelled routing set measured with a confusion matrix per source. Misroutes are logged, and the router is retrained monthly from traces.

Trade-off: pure LLM tool choice is flexible but slower and less predictable. Pure rules are brittle. The hybrid gets both."

**Follow-ups:**
- *What if sources disagree?* → Prefer the system of record (SQL/API over docs), show both with timestamps, and flag the discrepancy.
- *How do you handle 'clarify'?* → If required entities are missing, ask one targeted question instead of guessing.

**Red flags:**
- Sending every query to every source, which is slow and noisy.
- Letting an LLM write arbitrary SQL against production.

---

<a id="q14"></a>
## Q14. The agent produced valid JSON but chose the wrong tool. How do you handle wrong tool selection?
*Source: interview posts*

**What the interviewer is testing:** Understanding that schema validity isn't correctness, plus runtime detection and systematic fixes.

**Strong answer:**
"Schema validation only proves the call is *well-formed*. It says nothing about whether it's the *right* call. I handle this at three levels.

**1. Prevent (design time):**
- Make tools distinct: clear 'use when / don't use when' descriptions and no overlapping names.
- Keep fewer tools per agent. Route to a toolset first.
- Add few-shot examples for the confusable pairs I see in traces.

**2. Detect and contain (runtime):**
- **Preconditions in the handler**: `cancel_order` checks the order is cancellable and that the user intent was 'cancel'. If not, it returns `{error: "precondition_failed", why}`, and the model re-plans.
- **Read before write.** Side-effecting tools require a prior read of the entity in the same run (enforced by the orchestrator).
- **Intent/action consistency check** for risky tools: a small model or a rule verifies that the chosen tool matches the classified user intent. On a mismatch, block and ask.
- **Approval gates** on irreversible tools, so a wrong choice is caught before it matters.
- **Observation feedback.** If a read tool returns nothing relevant, the model usually self-corrects on the next step.

**3. Learn (offline):**
- Log every run with the tool sequence. Build a confusion matrix of expected vs chosen tool from labelled traces and user corrections.
- Fix the top confusions with descriptions or examples. If that plateaus, fine-tune or add a router.
- Add each incident to the regression eval set.

Example: our agent called `search_orders` when users pasted an order ID, because the descriptions overlapped. Tracing showed 11% misroutes. A regex pre-router ('ID pattern → `get_order`') plus rewritten descriptions brought it to 1.5%."

**Follow-ups:**
- *What if the wrong tool already caused a side effect?* → Compensating action (a saga pattern), audit log, alert. This is why irreversible actions sit behind approvals.
- *Can the LLM grade its own tool choice?* → A separate verifier call is better than self-grading, and only worth it for risky actions.

**Red flags:**
- "The JSON validated, so it's fine."
- Only fixing individual cases instead of measuring confusion rates.

---

<a id="q15"></a>
## Q15. Walk-through: "Plan a 3-day trip from Chennai to Coimbatore within ₹10,000." How does the agent handle the constraints?
*Source: interview posts*

**What the interviewer is testing:** End-to-end agent reasoning with hard constraints, verified by code rather than the LLM.

**Strong answer:**
"I'd first extract the constraints into **structured state**, because LLMs are bad at keeping a running budget in their head:

```json
{"origin":"Chennai","destination":"Coimbatore","days":3,"budget_inr":10000,
 "travellers":1,"dates":null,"preferences":[]}
```

`dates` is missing, so the agent either asks one question or assumes the next weekend and says so.

**Plan (plan-and-execute with re-planning):**
1. `search_transport(Chennai→CBE)`: train, bus, flight options with prices. Transport and hotel searches run **in parallel**.
2. `search_hotels(CBE, 2 nights)`
3. `search_attractions(CBE + Ooty/Isha day-trip)` with entry fees and distances.
4. **Combine options:** a deterministic optimizer, not the LLM, picks the combination that maximizes preference score subject to `total <= 10000`.
5. The LLM drafts the day-wise itinerary from the chosen items.
6. **Verify:** code recomputes the total from line items and checks dates and opening hours. If it fails, re-plan.

**Budget example:**

| Item | Cost (₹) |
|---|---|
| Train round trip (3AC) | 2,200 |
| Hotel, 2 nights | 3,600 |
| Local transport + Isha day trip | 1,500 |
| Food (3 days) | 1,800 |
| Entry fees | 400 |
| **Total** | **9,500** (₹500 buffer) |

**Re-planning:** if a flight is chosen and the total comes to ₹12,000, the verifier fails and the agent swaps to the train. If the hotel is sold out, it picks the next option. At most 2 re-plans, then it presents the best option and the trade-off: 'Closest fit is ₹10,400; here's what to drop.'

The key design point: **the LLM proposes and reasons, code enforces the constraints and does the arithmetic.** Prices come only from tool results, never from model memory, and each is cited with a timestamp because prices change. Booking would be a separate, approval-gated step."

**Follow-ups:**
- *Why not let the LLM sum the costs?* → Arithmetic and constraint satisfaction are unreliable in generation. Code is exact and auditable.
- *How do you handle stale prices?* → Re-verify with the live API right before booking. Quotes carry a timestamp and expiry.

**Red flags:**
- Letting the model invent prices from its training data.
- No verification step against the budget.

---

<a id="q16"></a>
## Q16. What makes a production agent more than an LLM (state, tools, validation, auth, observability)?
*Source: interview posts*

**What the interviewer is testing:** Whether you think in terms of the system (reliability, security, operability), not just a demo.

**Strong answer:**
"'User → LLM → response' is a demo. The goal in production isn't 'make it autonomous'. It's making it **reliable, observable, secure and controllable.** My checklist, grouped:

| Area | What it includes |
|---|---|
| **Orchestration & state** | Explicit state machine/graph (e.g. LangGraph), durable checkpoints, resumability |
| **Tools** | Typed schemas, validation, timeouts, retries with backoff, idempotency keys, rate limits |
| **Knowledge** | RAG with permissions-aware retrieval, freshness, citations |
| **Identity & access** | OAuth2/OIDC user auth, RBAC/ABAC, **tool-level permissions**, actions run as the user (not a super-account) |
| **Safety** | Input/output guardrails, prompt-injection defenses, PII redaction, HITL for critical actions |
| **Limits** | Max steps, token/cost budgets per run and per tenant, timeouts |
| **Observability** | Tracing per step (LangSmith/Langfuse/OpenTelemetry), token/cost metrics, tool error rates, latency P50/P95/P99 |
| **Quality** | Offline eval suite, online monitoring, regression tests on prompt/model changes |
| **Governance** | Audit logs (who, what, which tool, which args, approved by whom), versioned prompts/workflows, rollback |

A mental model I use: **LangChain → components, LangGraph → orchestration, RAG → knowledge, tools → actions, state → continuity, guardrails → control, evals and tracing → confidence.**

Concretely, for an HR helpdesk agent: SSO login; the agent can read only the employee's own records; `apply_leave` is a write tool with an idempotency key and a confirmation step; every call is traced; there's a ₹2 budget per conversation and 10 max steps; weekly eval of 300 golden queries; and an audit log retained for 1 year for compliance."

**Follow-ups:**
- *What would you build first with limited time?* → Tracing, hard limits and auth scoping, then evals, then the rest.
- *How do you make actions "run as the user"?* → Pass the user's delegated token (OAuth on-behalf-of) to downstream APIs so existing permissions apply.

**Red flags:**
- Treating security as "the system prompt says don't do bad things".
- No observability plan.

---

## Rapid-fire recap
- Agent = the LLM decides the next step in a loop with tools; workflow = code decides.
- The model only emits a structured tool call; your runtime validates, authorizes and executes.
- Tool descriptions are prompts: say what, when, when not, returns, formats.
- Errors go back to the model as observations, not exceptions.
- The model proposes stopping; the orchestrator enforces max steps, budget, timeout and repeat detection.
- ReAct adapts per step; plan-and-execute is cheaper and auditable; hybrid with re-plan triggers is the usual prod choice.
- Re-plan on failure or invalidated assumptions; keep completed work; cap re-plans.
- Never put identity or authority in model-supplied arguments.
- Constraints and arithmetic belong in code, not in generation.
- Schema-valid is not correct: add preconditions, intent checks and approvals for risky tools.
- Decision ladder: rules → single call → workflow → agent → multi-agent.
- Production agent = orchestration + tools + auth + guardrails + limits + tracing + evals + audit.
