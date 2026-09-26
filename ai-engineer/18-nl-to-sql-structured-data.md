# 18 — NL-to-SQL & Structured Data

**What interviewers probe here:** whether you know that "the query ran" is not the same as "the answer is right". Can you ground an LLM in business semantics, validate SQL before and after execution, and make database access safe?

[← Back to index](README.md)

1. [Syntactically correct SQL, wrong business answer](#q1)
2. [Design a production NL-to-SQL system](#q2)
3. [Making LLM-generated SQL safe](#q3)
4. [Evaluating NL-to-SQL](#q4)
5. [Schema linking for 1,000+ tables](#q5)
6. [SQL vs vector search vs both](#q6)

---

<a id="q1"></a>
## Q1. The generated SQL is syntactically correct but the business answer is wrong. How do you catch that before the user sees it?
*Source: interview posts*

**What the interviewer is testing:** Semantic validation. Do you know the typical ways correct-looking SQL is wrong?

**Strong answer:**

First, the common reasons valid SQL gives a wrong business answer:
- **Wrong definition:** "revenue" means net of refunds and excludes test accounts, but the model summed `orders.amount`.
- **Join fan-out:** joining orders to order_items and then summing order totals double-counts.
- **Missing default filters:** `is_deleted = false`, `status = 'completed'`, the current fiscal year, or the right timezone.
- **Wrong grain or time logic:** "last month" as rolling 30 days vs the calendar month; fiscal vs calendar year.
- **Ambiguous entity:** "customers in Hyderabad" means billing address vs shipping address.

How I catch it, in layers:

1. **Prevent with a semantic layer:** the model selects from governed metrics (dbt metrics / Cube / LookML-style definitions, e.g. `net_revenue`) instead of writing raw aggregations. This is the biggest win.
2. **Static checks on the SQL AST** (sqlglot): required filters present for each table, no many-to-many join before SUM without DISTINCT or pre-aggregation, date columns match the metric's definition.
3. **Execution sanity checks:**
   - Row count and null checks; results not empty or suspiciously large
   - **Reconciliation:** compare with a known aggregate (e.g. the total revenue for the month from the finance dashboard; flag if the difference exceeds 2%)
   - Range and plausibility checks (a conversion rate can't be 140%)
4. **Self-consistency:** generate 3 candidate SQLs; if the results disagree, don't answer confidently. Ask a clarifying question or show the assumptions.
5. **Transparency:** show the interpretation ("Net revenue, completed orders, calendar March 2026, IST") and the SQL, so the user can spot a wrong assumption.

Example: "Top 5 products by revenue last quarter" came back 18% inflated because of a join to a promotions table that duplicated rows. The fan-out check flagged `SUM` after a 1:N join, and the self-consistency variant using a subquery disagreed, so the system asked before answering. After adding the semantic layer, disagreement cases dropped from about 11% to 3%.

**Follow-ups:**
- *Which check gives the most value?* → The semantic layer plus reconciliation for key metrics.
- *What about latency?* → Static checks take milliseconds; run self-consistency only on high-stakes or low-confidence queries.

**Red flags:**
- "If the query executes, it's correct."
- Relying on the LLM to review its own SQL as the only check.

---

<a id="q2"></a>
## Q2. Design a production NL-to-SQL system.
*Source: interview posts*

**What the interviewer is testing:** End-to-end architecture with context, validation, safety and feedback.

**Strong answer:**

```
User question
  -> Intent + ambiguity check (clarify if needed)
  -> Schema linking: retrieve relevant tables/columns/metrics (vector + keyword over schema docs)
  -> Context builder: table DDL subset, column descriptions, sample values, join paths, business glossary, 3-5 similar verified Q->SQL examples
  -> LLM generates SQL (dialect-specific) + stated assumptions
  -> Validator: parse (sqlglot), policy checks, EXPLAIN / cost estimate
  -> Execute read-only with RLS, timeout, row limit
  -> Result checks (plausibility, reconciliation)
  -> LLM summarizes result + shows SQL/assumptions + chart
  -> Feedback: thumbs / corrections -> verified example store
```

Key decisions:
- **Metadata is the product.** Column descriptions, enums (`status IN ('C','P','X')` means completed/pending/cancelled), and verified join paths improve accuracy more than a bigger model.
- **Few-shot from a verified query library:** retrieve similar past questions with analyst-approved SQL. This is the single best accuracy lever I've seen, often worth 10–20 points.
- **Repair loop:** on an execution error, feed the error back once or twice, capped.
- **Clarification over guessing** for ambiguous terms.
- **Cost:** a small model for intent and linking, a strong model for generation, and caching for repeated questions.

Example numbers: on 400 internal eval questions, a baseline with the full schema scored 58% execution accuracy. With schema linking, descriptions and the verified examples it reached 81%, and 88% with the semantic layer for metric questions. P50 latency was 4 s, mostly generation plus the query itself.

**Follow-ups:**
- *Agent or pipeline?* → A pipeline with a bounded repair loop is more predictable; use an agent only for multi-step exploration.
- *How do you handle dialects?* → Pass the dialect in the prompt; sqlglot transpiles and validates.

**Red flags:**
- Dumping the whole schema into the prompt.
- No verified example library or feedback loop.

---

<a id="q3"></a>
## Q3. How do you make LLM-generated SQL safe?
*Source: interview posts*

**What the interviewer is testing:** Defense in depth at the database layer, not prompt instructions.

**Strong answer:**

The prompt saying "only write SELECT" is not a control. Real controls:

1. **Read-only DB role:** SELECT-only grants on an approved set of schemas and views, ideally against a read replica or warehouse, never the primary OLTP database.
2. **Row- and column-level security:** the query runs with the **end user's** identity or a session variable (`SET app.tenant_id`) that RLS policies use. Sensitive columns (salary, PII) are masked or excluded via secure views.
3. **AST allow-listing (sqlglot):** a single statement only; only `SELECT`/`WITH`; no DDL/DML, no `COPY`, `pg_sleep`, `LOAD_FILE`, or system catalogs; tables must be in the allow-list.
4. **Resource limits:** statement timeout (e.g. 30 s), automatic `LIMIT` (e.g. 10k rows), an EXPLAIN cost threshold, per-user concurrency and query quotas.
5. **No string-built values:** user-provided literals go in as bind parameters where the pipeline allows.
6. **Output controls:** mask PII in results and cap export size.
7. **Audit:** log user, question, SQL, rows returned.

```python
import sqlglot
from sqlglot import exp

ALLOWED = {"orders", "customers", "products"}
def validate(sql: str) -> None:
    stmts = sqlglot.parse(sql, read="postgres")
    if len(stmts) != 1 or not isinstance(stmts[0], (exp.Select, exp.Union)):
        raise ValueError("only single SELECT allowed")
    tables = {t.name.lower() for t in stmts[0].find_all(exp.Table)}
    if not tables <= ALLOWED:
        raise ValueError(f"disallowed tables: {tables - ALLOWED}")
```

Note: a `WITH ... SELECT` parses as a Select with a CTE, but data-modifying CTEs (Postgres `WITH x AS (DELETE ...)`) must also be rejected, which is one more reason the read-only role is the real backstop.

**Follow-ups:**
- *Why a replica?* → A runaway analytical query can't hurt production traffic.
- *Multi-tenant?* → RLS or a per-tenant schema, with the tenant from the auth context, never from the question.

**Red flags:**
- A regex check for "DROP".
- Running as the application's read/write user.

---

<a id="q4"></a>
## Q4. How do you evaluate NL-to-SQL?
*Source: interview posts*

**What the interviewer is testing:** Knowing that string-matching SQL is a bad metric, and building a representative eval set.

**Strong answer:**

Metrics:
- **Execution accuracy (primary):** run the generated and gold SQL and compare result sets (order-insensitive unless ORDER BY matters, with float tolerance). Different SQL can be equally correct.
- **Exact / component match:** a weak, secondary diagnostic only.
- **Test-suite accuracy:** run on multiple DB snapshots or perturbed data, to catch queries that match by coincidence.
- **Validity rate:** percentage that parse and execute.
- **Clarification quality:** for ambiguous questions, did it ask instead of guessing?
- **Latency and cost** per question.
- **Business-level:** user acceptance rate, analyst corrections, repeat-question rate.

Dataset: 300–500 real user questions sampled from logs, stratified by difficulty (single table, joins, time logic, nested, ambiguous), with analyst-verified gold SQL. Add a slice of "unanswerable with this schema" questions. Public benchmarks (Spider, BIRD) are for model screening only; they don't reflect your schema.

Process: run in CI on every prompt, model or metadata change; break results down by category; review failures and tag the cause (linking error, join error, filter missing, aggregation, dialect).

Example: we found that 40% of failures were "time logic" (fiscal quarter). One glossary entry plus two verified examples fixed most of them. Category breakdowns tell you what to fix; a single score doesn't.

**Follow-ups:**
- *Result comparison when columns differ in order or name?* → Compare values by column-set matching; normalize types.

**Red flags:**
- Using BLEU or exact SQL match as the main metric.
- Only evaluating on Spider.

---

<a id="q5"></a>
## Q5. The schema has 1,000+ tables. How do you do schema linking?
*[Added]*

**What the interviewer is testing:** Retrieval over metadata, and pruning context.

**Strong answer:**

You can't put 1,000 tables of DDL into the prompt; it costs too much and accuracy drops. Schema linking is a retrieval problem:

1. **Build a schema catalog:** a document per table (name, description, columns with descriptions, sample values, row count, owner, popular joins) and per column for important ones. Descriptions can be LLM-generated first, then human-reviewed.
2. **Hybrid retrieval** over the catalog: BM25 catches exact names like `gst_invoice`; embeddings catch synonyms ("tax bill"). Also match question terms against **column value indexes** (e.g. "Bengaluru" appears in `customers.city`).
3. **Rerank and prune:** take the top 20 tables, then have an LLM or cross-encoder select 3–8. Include their FK neighbors so joins are possible.
4. **Join graph:** precompute FK and usage-based join paths from the query logs; give the model the path, not just the tables.
5. **Domain routing:** first classify into a domain (finance, sales, HR), then retrieve within it.
6. **Usage priors:** tables used in verified queries rank higher; deprecated tables are excluded.

Measure linking separately: **table recall@k**, meaning whether the gold tables are in the selected set. If linking recall is 85%, generation accuracy can't exceed that.

Example: 1,400 tables in a warehouse. Domain routing plus hybrid retrieval over the catalog gave table recall@8 of 94%, and the prompt shrank from about 120k tokens to 6k.

**Follow-ups:**
- *Wide tables with 300 columns?* → Column-level retrieval; include only the relevant columns plus the keys.

**Red flags:**
- "Use a long-context model and pass everything."

---

<a id="q6"></a>
## Q6. When do you route to SQL vs vector search vs both?
*[Added]*

**What the interviewer is testing:** Picking the right retrieval modality for the question type.

**Strong answer:**

| Question type | Route | Example |
|---|---|---|
| Aggregations, counts, filters, rankings over structured data | SQL | "Total sales by region in Q2" |
| Explanations, policies, unstructured text | Vector / hybrid RAG | "What is our refund policy for damaged items?" |
| Exact lookups by ID | SQL or keyword | "Status of order 88213" |
| Mixed | Both, then compose | "Which of our top 10 customers by revenue have open complaints mentioning delays?" (SQL for the top 10, text search over their tickets) |

Never answer aggregation questions with vector RAG. Retrieving 5 chunks can't give you a correct sum over 2 million rows.

Routing implementation: a lightweight classifier (a fine-tuned small model or an LLM with structured output) returns `{route: sql|rag|both, confidence}`. Low confidence goes to "both" or a clarifying question. For "both", an orchestrator runs SQL first to get entity IDs, then filters the vector search by those IDs via metadata.

Evaluation: build a routing eval set (e.g. 200 labelled questions) and track routing accuracy separately. It's often the hidden cause of bad answers.

**Follow-ups:**
- *Can text-to-SQL use semantic search inside?* → Yes. Some DBs support vector columns (pgvector), so hybrid SQL plus similarity in one query.

**Red flags:**
- Embedding CSV rows and asking RAG for totals.

---

## Rapid-fire recap

- Valid SQL is not a correct answer; watch definitions, join fan-out, default filters and time logic.
- A semantic layer (governed metrics) is the biggest accuracy and consistency lever.
- A verified Q→SQL example library, retrieved as few-shot, is the next biggest.
- Validate with an AST parser (sqlglot), not regex.
- Safety = read-only role + replica + RLS + allow-list + timeouts + row limits.
- The tenant/user comes from auth context, never from the question.
- Execution accuracy on a real-question eval set is the primary metric.
- Measure schema-linking recall separately; it caps overall accuracy.
- Show the interpretation and SQL to the user; ask when the question is ambiguous.
- Aggregations go to SQL, explanations to RAG, mixed questions to both.
