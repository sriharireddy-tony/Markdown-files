# 21 — Domain Scenarios: AI in Supply Chain

**What interviewers probe here:** can you connect model outputs to business decisions? Interviewers look for how you validate recommendations, where human judgment has to stay, how you put guardrails on autonomous agents that spend money, and how you measure real value. The same reasoning applies in any domain (finance, HR, healthcare), so the framing below transfers.

[← Back to index](README.md)

1. [Model says +30% inventory, Finance rejects](#q1)
2. [Predicted demand spike planners don't believe](#q2)
3. [Controls for autonomous procurement](#q3)
4. [Forecast accuracy up, inventory not improved](#q4)
5. [Cheaper but less reliable supplier](#q5)
6. [Mathematically optimal, operationally broken](#q6)
7. [Good in one region, poor in another](#q7)
8. [Decisions never fully delegated to AI](#q8)
9. [ROI beyond hours saved](#q9)
10. [AI vs business strategy: who decides](#q10)

> A reusable framing for all ten: **Data → Model → Decision → Execution → Outcome.** Find which link is broken before blaming the model.

---

<a id="q1"></a>
## Q1. The AI recommends increasing inventory 30% but Finance rejects it. How do you validate the model?
*Source: interview posts*

**What the interviewer is testing:** Treating disagreement as a validation exercise, and making the recommendation explainable in financial terms.

**Strong answer:**

I wouldn't argue "the model is right". I'd make the recommendation testable.

1. **Understand Finance's objection.** Is it cash and working capital, storage cost, obsolescence risk, or distrust of the forecast? Each points to a different check.
2. **Decompose the recommendation.** A 30% increase comes from a forecast, a service-level target and lead-time variability. Which driver moved? For example: "the forecast rose 12%, lead-time variability doubled after the supplier change, and the safety stock formula amplified it."
3. **Validate the inputs.** Is the demand forecast biased? Check its historical bias and MAPE by SKU. Is the lead-time data correct, or is one delayed shipment an outlier?
4. **Backtest the policy, not just the forecast.** Simulate the last 12 months: with this policy, what would stockouts, holding cost and working capital have been, compared with what actually happened?
5. **Quantify the trade-off in money.** "+30% inventory = ₹4.2 Cr more working capital and ₹35 L per year holding cost. It reduces the expected stockout loss by ₹60 L per year at a 97% service level." Now Finance is evaluating a business decision instead of trusting a black box.
6. **Offer a controlled test.** Apply the policy to 20% of SKUs or one DC for 8 weeks, with a matched control group.

Possible outcomes: the model is right (the decision is data-backed), it's partly right (keep the increase on high-margin, high-variability SKUs only), or it's wrong (e.g. lead-time data was corrupted, so fix the pipeline).

**Follow-ups:**
- *What if Finance still says no?* → The constraint is legitimate. Add a working-capital cap as an optimization constraint, and let the model optimize within it.

**Red flags:**
- "The model's accuracy is 92%, so Finance should trust it."
- No translation of the recommendation into cash and risk.

---

<a id="q2"></a>
## Q2. The AI predicts a demand spike that planners don't believe. Model or human experience?
*Source: interview posts*

**What the interviewer is testing:** Using calibrated evidence rather than choosing a side, and designing human-AI collaboration.

**Strong answer:**

Neither blindly. I'd resolve it with **evidence and track record**.

- **Ask why.** What's driving the spike: a promotion flag, seasonality, a social signal, or a data glitch (a duplicated order feed)? Explainability (feature attributions, similar past periods) turns the debate into a concrete conversation.
- **Ask planners why not.** They often have information the model lacks: a competitor launch, a known one-off bulk order last year, a channel change. That's missing-feature information, and it's valuable.
- **Check track records.** How accurate have past model-predicted spikes been, compared with planner overrides? Forecast value added (FVA) analysis measures whether human overrides improve or worsen accuracy. Many companies find overrides help on some product categories and hurt on others.
- **Decide by asymmetric cost.** If under-stocking costs 5x over-stocking (a high-margin product with a short shelf life), hedge towards the forecast. Use partial pre-positioning, or flexible supplier capacity options instead of committing all volume.
- **Close the loop.** Record the override and its reason, then evaluate afterwards. Over time you learn where to trust whom.

Example answer: "I'd split the difference with an option: order 50% of the incremental volume now, and secure supplier capacity for the rest with a 2-week trigger based on early sell-through signals."

**Follow-ups:**
- *How do you prevent override bias?* → Require reason codes and measure FVA per planner and per category.

**Red flags:**
- "Always trust the model." or "Always trust humans."

---

<a id="q3"></a>
## Q3. An AI agent auto-places purchase orders. What controls do you put around autonomous procurement?
*Source: interview posts*

**What the interviewer is testing:** Agent guardrails for financial actions, enforced in code.

**Strong answer:**

I'd apply autonomy tiers plus hard controls:

**Autonomy tiers**

| Tier | Condition | Mode |
|---|---|---|
| Auto | Repeat order, approved supplier, contracted price, within ±15% of the usual quantity, < ₹2 L | Auto-place, logged |
| Review | New quantity pattern, ₹2–20 L, price deviation > 3% | Human approval of the exact PO |
| Blocked | New supplier, off-contract item, > ₹20 L, sanctioned or flagged vendor | Agent can only draft |

**Hard controls (code, not prompt)**
- Approved vendor list and contract price check against the ERP
- Budget checks against cost-centre limits; daily and monthly spend caps per agent
- Quantity sanity checks: vs forecast, MOQ, warehouse capacity, shelf life
- Duplicate detection and **idempotency keys** (the same PO request can't be created twice on retry)
- Segregation of duties: the agent that recommends can't also approve and receive
- Four-eyes approval above thresholds
- An audit trail: inputs, forecast version, reasoning summary, approver
- A kill switch and anomaly alerts (e.g. PO volume 3x the daily norm)

**Rollout:** shadow mode first (the agent proposes, humans place; measure agreement), then auto-mode for the lowest-risk tier, expanding as the error rate stays below target.

Example: in shadow mode for 6 weeks, the agent matched buyer decisions on 91% of repeat orders. Most disagreements came from supplier stock-out information that wasn't in the ERP. We added a supplier-availability API before enabling auto-mode.

**Follow-ups:**
- *What if the agent is manipulated (a supplier email with an injected instruction)?* → The agent can't change the vendor, bank details or price; those come from master data only.

**Red flags:**
- "The agent will be instructed to stay within budget."

---

<a id="q4"></a>
## Q4. Forecast accuracy improved but inventory didn't. What explains the gap?
*Source: interview posts*

**What the interviewer is testing:** Understanding that a better model doesn't automatically mean a better decision or better execution.

**Strong answer:**

Forecast accuracy is an intermediate metric. The chain is forecast → planning parameters → orders → supplier execution → inventory. Common breaks:

1. **The forecast isn't used.** Planners override it, or the ERP still uses old min/max parameters that were never updated from the new forecast.
2. **The wrong accuracy.** Aggregate MAPE improved, but the SKU-location-week level (where ordering happens) didn't, or the gains came from high-volume stable SKUs that were already fine. The errors that drive inventory are on the long-tail, volatile SKUs.
3. **Bias vs error.** MAPE improved but the forecast is still **biased high**, so stock keeps building up.
4. **Safety stock drivers.** Inventory is dominated by lead-time variability, MOQs, lot sizes and supplier reliability, not forecast error.
5. **Policy constraints.** Service-level targets set conservatively at 99%, or batching, container fill rules and volume discounts drive ordering.
6. **Time lag.** Inventory adjusts over one or more replenishment cycles; it may be too early to see the effect.
7. **Measurement.** Mix change, new product launches or channel shifts confound the before/after comparison.

What I'd do: measure the **right** metrics (bias, weighted MAPE at the decision level, forecast value added), trace a sample of SKUs end to end from forecast to PO, and connect the forecast to the inventory policy (dynamic safety stock).

**Follow-ups:**
- *Which KPI would you report?* → Inventory days, service level/fill rate, stockouts and obsolescence together, not forecast accuracy alone.

**Red flags:**
- "Retrain the model to improve accuracy further."

---

<a id="q5"></a>
## Q5. The AI recommends a cheaper but less reliable supplier. How do you evaluate that?
*Source: interview posts*

**What the interviewer is testing:** Total-cost and risk-adjusted thinking rather than unit price.

**Strong answer:**

Unit price is not total cost. I'd evaluate the **total landed, risk-adjusted cost**:

- Unit price, freight, duties and payment terms
- **Reliability costs:** on-time delivery rate, lead-time variability (which drives higher safety stock and working capital), defect rate (rework, returns, quality inspection cost)
- **Stockout exposure:** P(late) × impact (lost margin, line stoppage, penalty clauses)
- Switching cost: qualification, onboarding, tooling
- Strategic risk: concentration, geopolitical, financial health, ESG/compliance

Worked example:

| | Supplier A (current) | Supplier B (AI pick) |
|---|---|---|
| Unit price | ₹100 | ₹92 |
| On-time rate | 97% | 85% |
| Extra safety stock cost | — | +₹2.1 L/yr |
| Expected stockout cost | ₹1.0 L/yr | ₹6.5 L/yr |
| Savings on 1 L units | — | ₹8 L/yr |
| **Net** | baseline | **≈ -₹0.6 L/yr (worse)** |

Also check whether the model even includes these costs. If its objective function is price only, the model isn't wrong; the **objective is incomplete**.

A balanced option: **dual sourcing**. Keep A for critical volume (70%) and trial B at 30% with a scorecard, which builds reliability data while limiting risk.

**Follow-ups:**
- *How would you fix the model?* → Add reliability features and risk costs into the optimization objective.

**Red flags:**
- Comparing unit price only.

---

<a id="q6"></a>
## Q6. The plan is mathematically optimal but causes operational problems. What do you do?
*Source: interview posts*

**What the interviewer is testing:** Humility about models, and iterating on constraints with operators.

**Strong answer:**

"Optimal but unworkable" almost always means **missing constraints or a wrong objective**. The model optimized a simplified world.

1. **Go to the floor.** Talk to warehouse, transport and production teams: what exactly breaks? Examples: too many small shipments, schedule changes every day, dock capacity exceeded, shift patterns ignored.
2. **Classify the gap:**
   - Hidden hard constraints (dock slots, labor per shift, truck sizes, changeover times)
   - Soft preferences (plan stability, minimum batch sizes, driver routes)
   - Data errors (wrong capacity values in master data)
3. **Encode them:** add the hard constraints, and penalties for soft ones. Add a **plan-stability penalty** so the plan doesn't swing daily ("nervousness"), and freeze windows (no changes within 48 h of execution).
4. **Accept slightly lower "optimal" cost for feasibility.** A plan that is 2% worse on paper but executable beats a theoretical optimum that people ignore.
5. **Co-design:** planners review and adjust in a UI; their adjustments are logged as signals of missing constraints.
6. **Measure realized outcomes** (actual cost, OTIF, overtime) rather than plan objective values.

Example: a route optimizer cut modeled distance by 11%, but drivers rejected it because of unloading windows at retail stores. Adding time windows and a max of 2 route changes per week kept a 7% gain, and adoption went from 30% to 90%.

**Follow-ups:**
- *Who owns constraints?* → Operations owns the definitions; the data and AI team encodes them and keeps them versioned.

**Red flags:**
- "Operations needs to adapt to the optimal plan."

---

<a id="q7"></a>
## Q7. The model performs well in one region and poorly in another. How do you investigate?
*Source: interview posts*

**What the interviewer is testing:** Systematic segment-level diagnosis.

**Strong answer:**

I'd compare the regions across four areas:

1. **Data quality:** missing or late data feeds, different ERP systems, unit or currency mismatches, data coverage (fewer years of history), and a different stock-out rate. Stockouts censor demand, so sales look lower than true demand.
2. **Distribution differences:** different seasonality (festival calendars differ: Diwali vs Onam vs Pongal timing), weather, promotions, channel mix (modern trade vs kirana), and product mix (more long-tail SKUs).
3. **Training representation:** was the poor region under-represented in training? Does one global model fit regional behavior, or do we need regional features or models?
4. **Evaluation artifacts:** is the error metric comparable? MAPE explodes for low-volume regions; use WAPE or MASE. Also check whether the evaluation periods differ.

Steps: segment-level error breakdown (region × category × volume band), residual analysis over time, feature importance by region, and data-lineage checks. Then **fix the cause**: add region-specific features (local holiday calendars), hierarchical models or regional fine-tuning, fix the data feeds, or correct for censored demand.

Example: the South region showed WAPE of 38% vs 18% in the North. The main cause was a regional festival calendar missing from features, plus a distributor feed that arrived with a 5-day lag. Fixing both brought it to 22%.

**Follow-ups:**
- *One model or many?* → Start global with regional features (more data sharing), and split only when the residuals justify it.

**Red flags:**
- "Train a separate model for that region" without diagnosis.

---

<a id="q8"></a>
## Q8. Which supply-chain decisions should never be fully delegated to an AI agent?
*Source: interview posts*

**What the interviewer is testing:** Principled boundaries on autonomy.

**Strong answer:**

I'd use a principle rather than a fixed list: **don't fully delegate decisions that are high-impact, irreversible, novel, or require accountability, ethics or relationships.** Concretely:

- **Strategic sourcing:** selecting or terminating strategic suppliers, and long-term contracts, where relationships, negotiation and geopolitics matter.
- **Large or irreversible financial commitments:** capital purchases, large forward buys, and hedging above a threshold.
- **Crisis response:** disruptions such as floods, strikes or pandemics. There's no historical data, so the model is extrapolating.
- **Safety, quality and compliance:** product recalls, quality release of regulated goods (pharma, food), sanctions and export control.
- **Ethical/ESG trade-offs:** sourcing from risky labor regions, sustainability commitments.
- **People-impacting decisions:** workforce scheduling that affects livelihoods, and supplier penalties.
- **Master data changes:** vendor bank details, which are a fraud vector.

AI can still **prepare** these decisions: options, scenario analysis, risk scoring. A human decides and is accountable.

What **can** be delegated with guardrails: routine replenishment, low-value repeat POs, slotting, routine carrier selection, and exception triage.

**Follow-ups:**
- *Can boundaries move?* → Yes, with evidence: shadow mode, performance history, and a governance review.

**Red flags:**
- "With enough data, everything can be automated."

---

<a id="q9"></a>
## Q9. How do you calculate ROI beyond "hours saved"?
*Source: interview posts*

**What the interviewer is testing:** Business-outcome measurement with a proper baseline and costs.

**Strong answer:**

Hours saved is the weakest metric; hours only matter if they're redeployed. I'd measure business outcomes against a **counterfactual baseline** (control group or pre/post with seasonality adjustment):

**Value levers**
- **Working capital:** inventory reduction × cost of capital (e.g. ₹10 Cr less inventory × 12% = ₹1.2 Cr/yr)
- **Revenue protection:** fewer stockouts × margin (lost-sales estimates)
- **Cost:** lower expediting and premium freight, less obsolescence/write-offs, better procurement prices
- **Service:** OTIF/fill-rate improvement (retention, fewer penalty clauses)
- **Risk:** avoided disruption losses (expected value), fewer fraud or compliance incidents
- **Decision speed:** faster response to disruptions (days to hours), valued through avoided losses

**Costs (full TCO):** build, licenses and LLM/API usage, infrastructure, data engineering, change management and training, ongoing monitoring and model maintenance, and human review time.

Method: define KPIs and the baseline **before** launch; run a controlled pilot (region or SKU-group A/B); attribute carefully, since other initiatives happen simultaneously; report ranges, not point estimates; and track adoption. Low adoption means zero value.

Example: the pilot DC vs 2 matched control DCs over 16 weeks showed 9% lower inventory, stockouts down from 4.1% to 3.2%, and premium freight down 22%. Annualized net value was ₹2.8 Cr against a ₹0.9 Cr total annual cost, about 3x ROI.

**Follow-ups:**
- *What if the effect is small and noisy?* → Extend the pilot, use difference-in-differences, and say so honestly.

**Red flags:**
- ROI = hours saved × salary, with no baseline.

---

<a id="q10"></a>
## Q10. The AI's recommendation conflicts with business strategy. Who decides?
*Source: interview posts*

**What the interviewer is testing:** Governance maturity, and understanding that the AI optimizes the objective it's given.

**Strong answer:**

**The accountable business owner decides. AI informs; it doesn't govern.** But the conflict itself is a useful signal:

- **The AI optimizes the objective it was given.** If it recommends cutting a low-margin product line the company is deliberately investing in (market entry, a customer relationship), the objective doesn't include strategic value. Encode strategy as constraints or weights ("maintain 98% service for strategic accounts regardless of margin").
- **Or the strategy may be outdated.** If the AI repeatedly shows that a strategic choice costs ₹X Cr per year, leadership should see that number and decide consciously. That's AI adding value.

Governance I'd set up:
- A **RACI**: model owner (data/AI team) is responsible for correctness; the business owner is accountable for decisions.
- A documented override process: reason codes, visible to leadership, reviewed monthly.
- A decision-rights matrix: which decisions AI can automate, recommend, or only analyze.
- Escalation for large conflicts (above a value threshold) to a steering committee.

**Follow-ups:**
- *What if overrides are consistently wrong?* → Show the evidence (FVA and outcome tracking) and adjust the decision rights.

**Red flags:**
- "The AI is objective, so it should decide."

---

## Rapid-fire recap

- Diagnose along Data → Model → Decision → Execution → Outcome before blaming the model.
- Translate recommendations into cash, risk and service-level numbers.
- Resolve model-vs-human disagreements with evidence: track records and forecast value added.
- Autonomous procurement needs tiers, hard caps, approved-vendor checks, idempotency and a kill switch.
- Forecast accuracy is not inventory; check bias, decision-level granularity and policy parameters.
- Compare suppliers on total landed, risk-adjusted cost, not unit price.
- "Optimal but unworkable" means missing constraints; add them, plus plan stability.
- For regional gaps, segment the errors and check data feeds, local calendars and metric choice.
- Keep strategic, irreversible, novel and ethical decisions human.
- ROI needs a counterfactual baseline and full TCO.
- The business owner decides; the AI optimizes the objective you gave it.
