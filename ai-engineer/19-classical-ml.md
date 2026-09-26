# 19 — Classical ML Fundamentals

**What interviewers probe here:** AI-engineer rounds still include classical ML checks, especially metrics on imbalanced data. Interviewers want to see that you pick metrics from the business cost of errors, and that you can reason about overfitting, leakage and validation. These same skills apply when you evaluate LLM classifiers and guardrails.

[← Back to index](README.md)

1. [Classification vs regression](#q1)
2. [Why accuracy can be misleading](#q2)
3. [Metrics for imbalanced datasets](#q3)
4. [Handling imbalanced datasets](#q4)
5. [Micro vs macro vs weighted F1](#q5)
6. [Loss functions and gradient descent](#q6)
7. [Bias-variance, overfitting, regularization](#q7)
8. [Data leakage](#q8)
9. [Cross-validation](#q9)

---

<a id="q1"></a>
## Q1. Classification vs regression.
*Source: interview posts*

**What the interviewer is testing:** Clean fundamentals, and whether you can map a business problem to the right formulation.

**Strong answer:**

- **Classification** predicts a discrete label: spam or not, churn yes/no, intent class. The model usually outputs probabilities, and a **threshold** turns them into decisions. Losses: cross-entropy / log loss. Metrics: precision, recall, F1, ROC-AUC, PR-AUC, log loss.
- **Regression** predicts a continuous value: delivery time, price, demand. Losses: MSE (penalizes large errors), MAE (robust to outliers), Huber (a mix). Metrics: MAE, RMSE, MAPE (breaks near zero), R².

The formulation is a design choice:
- "Will the customer churn?" (classification) vs "days until churn" (regression / survival analysis).
- Ordinal targets (ratings 1–5) can be treated either way; ordinal regression respects the ordering.
- Sometimes regression plus a threshold is better: predict the fraud amount, then act on the expected loss.

In GenAI systems this still matters: intent routing, guardrails and relevance grading are classification problems, and confidence scores need calibration.

Example: for a delivery ETA feature, MAE of 6 minutes was the headline metric, but the business cared about "late beyond promise". So we also tracked the percentage of orders with error above 15 minutes, which is effectively a classification view of a regression model.

**Follow-ups:**
- *Is logistic regression regression?* → It models log-odds linearly but is used for classification.
- *Which loss for regression with outliers?* → MAE or Huber.

**Red flags:**
- Not mentioning thresholds or probabilities for classification.

---

<a id="q2"></a>
## Q2. Why can accuracy be misleading?
*Source: interview posts*

**What the interviewer is testing:** Class imbalance and unequal error costs.

**Strong answer:**

Accuracy counts all errors equally and is dominated by the majority class.

Example: fraud rate is 0.5%. A model that always predicts "not fraud" scores **99.5% accuracy** and catches zero fraud, which makes it useless.

Accuracy also hides:
- **Asymmetric costs:** missing a cancer case (FN) is far worse than a false alarm (FP).
- **Per-class failure:** 95% overall accuracy can hide 20% recall on the class you care about.
- **Threshold dependence:** accuracy at 0.5 says nothing about the model's ranking quality.
- **Distribution shift:** if the class mix changes in production, accuracy moves even when the model hasn't changed.

What I use instead: the confusion matrix first, then precision and recall per class, then PR-AUC for rare positives, and finally a **cost-weighted metric** (e.g. expected ₹ loss = FN × ₹5,000 + FP × ₹50) to choose the threshold.

**Follow-ups:**
- *When is accuracy fine?* → Balanced classes with roughly equal error costs.
- *Balanced accuracy?* → The mean of per-class recall; better than accuracy for imbalance.

**Red flags:**
- Reporting only accuracy on a skewed dataset.

---

<a id="q3"></a>
## Q3. Which metrics do you use for imbalanced datasets (precision, recall, F1, ROC-AUC vs PR-AUC)?
*Source: interview posts*

**What the interviewer is testing:** Knowing what each metric means and when ROC-AUC is over-optimistic.

**Strong answer:**

- **Precision** = TP / (TP + FP): of those I flagged, how many were right? It matters when false alarms are costly (manual review queues).
- **Recall** = TP / (TP + FN): of all real positives, how many did I catch? It matters when misses are costly (fraud, disease, safety).
- **F1** = the harmonic mean of the two, a single number when both matter. **F-beta** weights recall (β>1) or precision (β<1).
- **ROC-AUC:** the probability that a random positive ranks above a random negative. It uses the FPR = FP / (FP + TN). With huge TN counts, FPR stays tiny even with many false positives, so **ROC-AUC looks great on imbalanced data**.
- **PR-AUC (average precision):** focuses on the positive class. Its baseline equals the positive rate (e.g. 0.005), so it shows real difficulty.

Worked illustration: 100,000 transactions, 500 fraud. At some threshold the model catches 400 frauds (recall 0.8) with 2,000 false positives. FPR = 2,000 / 99,500 ≈ 2%, which looks excellent on a ROC curve. But precision = 400 / 2,400 = **16.7%**, so reviewers waste 5 of every 6 reviews. PR-AUC would expose this; ROC-AUC hides it.

Then pick the operating point: choose the threshold from business constraints, e.g. "reviewers can handle 1,000 alerts per day", which means maximizing recall at a fixed alert volume or at a minimum precision.

**Follow-ups:**
- *Calibration?* → If the scores drive expected-cost decisions, check reliability curves and the Brier score; calibrate with Platt scaling or isotonic regression.
- *MCC?* → Matthews correlation coefficient is a balanced single metric that uses all four confusion-matrix cells.

**Red flags:**
- "ROC-AUC is 0.97, so the model is great" on 0.5% positives.

---

<a id="q4"></a>
## Q4. How do you handle an imbalanced dataset?
*Source: interview posts*

**What the interviewer is testing:** A toolkit, and the judgment of when each tool helps.

**Strong answer:**

I work through the options in order:

1. **Fix evaluation first:** stratified splits, PR-AUC or recall at a fixed precision, and a confusion matrix. Often the "imbalance problem" is really a metric problem.
2. **Threshold tuning:** the cheapest and often most effective fix. A default 0.5 threshold is arbitrary; tune it on validation data against the business cost.
3. **Class weights / cost-sensitive loss:** `class_weight="balanced"` or `scale_pos_weight` in XGBoost, or focal loss in deep learning. This keeps all data.
4. **Resampling:** random undersampling of the majority (fast, loses information), oversampling or **SMOTE** for the minority (risk of overfitting and synthetic noise). **Only resample the training fold**, never validation or test.
5. **More or better minority data:** active learning, labelling more positives, weak supervision, synthetic data (LLM-generated examples work for text classes, but validate them).
6. **Model/formulation:** anomaly detection when positives are extremely rare or evolving; two-stage models (a high-recall filter, then a precise classifier).

Trade-off: resampling and weights distort the predicted probabilities. If downstream logic needs calibrated probabilities, recalibrate afterwards.

Example: a support-ticket escalation classifier with 3% positives. Class weights plus threshold tuning raised recall from 0.41 to 0.78 at precision 0.60. SMOTE added nothing on top and slightly hurt calibration, so we dropped it.

**Follow-ups:**
- *Why not SMOTE before the split?* → Synthetic points derived from test neighbors leak into training; that is data leakage.

**Red flags:**
- Jumping straight to SMOTE without fixing metrics or thresholds.
- Resampling the test set.

---

<a id="q5"></a>
## Q5. Micro F1 vs macro F1 (and weighted F1).
*Source: interview posts*

**What the interviewer is testing:** Averaging semantics in multi-class problems, with a worked example.

**Strong answer:**

- **Micro F1:** pool the TP/FP/FN across all classes, then compute F1. Every **sample** counts equally, so the majority classes dominate. In single-label multi-class classification, **micro F1 = accuracy**.
- **Macro F1:** compute F1 per class, then take the unweighted mean. Every **class** counts equally, so it exposes poor performance on rare classes.
- **Weighted F1:** per-class F1 averaged by support (class frequency). It sits close to micro and still hides minority failures.

Worked example: an intent classifier on 1,000 queries with 3 classes.

| Class | Support | TP | FP | FN | F1 = 2TP/(2TP+FP+FN) |
|---|---|---|---|---|---|
| billing | 900 | 880 | 40 | 20 | 1760/1820 = **0.967** |
| refund | 80 | 50 | 15 | 30 | 100/145 = **0.690** |
| legal_complaint | 20 | 5 | 10 | 15 | 10/35 = **0.286** |

- **Micro F1** = total TP / N = 935/1000 = **0.935**
- **Macro F1** = (0.967 + 0.690 + 0.286) / 3 = **0.648**
- **Weighted F1** = 0.9(0.967) + 0.08(0.690) + 0.02(0.286) ≈ **0.931**

Micro and weighted say "93% — great". Macro says the model is failing on legal complaints, which here are the most important class to route correctly.

When to use which: macro when all classes matter equally, especially rare ones. Micro for overall throughput. Always also show the per-class table.

**Follow-ups:**
- *Multi-label?* → Micro and macro are both defined; micro is no longer equal to accuracy.
- *Why harmonic mean?* → It punishes imbalance between P and R; F1 is high only if both are high.

**Red flags:**
- Not knowing that micro F1 equals accuracy in single-label multi-class.

---

<a id="q6"></a>
## Q6. Neural network basics: loss functions and gradient descent.
*Source: interview posts*

**What the interviewer is testing:** You understand what training actually does, which also underpins fine-tuning discussions.

**Strong answer:**

- A neural network is layers of linear transforms followed by non-linear activations (ReLU, GELU). Without the non-linearity, stacked layers collapse into a single linear map.
- The **loss function** measures how wrong the predictions are: cross-entropy for classification and for LLM next-token prediction; MSE/MAE for regression; contrastive losses (InfoNCE) for embedding models.
- **Backpropagation** uses the chain rule to compute the gradient of the loss with respect to every weight.
- **Gradient descent** updates the weights: `w = w - lr * grad`. Variants:
  - **SGD / mini-batch:** gradients estimated on batches (e.g. 32–4096 samples); the noise helps generalization.
  - **Momentum / Adam / AdamW:** adaptive per-parameter step sizes; AdamW (decoupled weight decay) is the default for transformers.
- **Learning rate** is the most important hyperparameter. Too high diverges; too low wastes compute. Transformers use **warmup then cosine/linear decay**.
- Stability tools: normalization (LayerNorm/RMSNorm), residual connections, gradient clipping, mixed precision (bf16).

Connection to LLMs: pretraining minimizes next-token cross-entropy; perplexity = exp(loss). LoRA fine-tuning runs the same gradient descent, but only on small low-rank adapter matrices.

**Follow-ups:**
- *Vanishing gradients?* → Addressed by ReLU-family activations, residual connections and normalization.
- *Why does the loss plateau?* → Learning rate issues, data quality or model capacity; check the train/val curves.

**Red flags:**
- Confusing backprop (computing gradients) with gradient descent (updating weights).

---

<a id="q7"></a>
## Q7. Bias-variance trade-off, overfitting and regularization.
*[Added]*

**What the interviewer is testing:** Diagnosing a model from its learning curves.

**Strong answer:**

- **Bias:** error from an overly simple model (underfitting). Train and validation error are both high.
- **Variance:** error from sensitivity to the training data (overfitting). Train error is low and validation error is much higher.
- The goal is minimum validation/test error, not training error.

Diagnosis from learning curves:
- A big train–val gap means overfitting: get more data, regularize, or use a simpler model.
- Both high and close together means underfitting: use a bigger model, better features, or train longer.

Regularization options: L2 (weight decay), L1 (sparsity), dropout, early stopping, data augmentation, limiting tree depth or min samples per leaf, and ensembling (bagging reduces variance). In fine-tuning LLMs: fewer epochs (1–3), low LoRA rank, and watching validation loss; validation loss rising after epoch 2 means stop.

Modern nuance: very large models show "double descent", where overparameterized models can still generalize. Practical monitoring stays the same: held-out validation.

Example: a gradient-boosting churn model had train AUC 0.99 and val AUC 0.78. Reducing max_depth from 12 to 5, subsampling at 0.8 and early stopping gave train 0.88 and val 0.84.

**Follow-ups:**
- *L1 vs L2?* → L1 drives weights to zero (feature selection); L2 shrinks them smoothly.

**Red flags:**
- Judging a model by its training accuracy.

---

<a id="q8"></a>
## Q8. What is data leakage, and how do you catch it?
*[Added]*

**What the interviewer is testing:** The number one cause of models that "worked offline" and failed in production.

**Strong answer:**

Leakage is when training uses information that won't be available at prediction time, so offline metrics are inflated.

Types:
- **Target leakage:** a feature derived from the outcome. `refund_issued` used to predict a complaint; `days_since_cancellation` used to predict churn.
- **Train/test contamination:** scaling, SMOTE or imputation fitted on the full dataset before splitting; duplicate or near-duplicate rows across splits.
- **Temporal leakage:** random splits on time-series data, so the model sees the future.
- **Group leakage:** the same customer or patient in both train and test.
- **LLM-era leakage:** benchmark questions present in pretraining data; eval examples reused as few-shot examples or used in prompt tuning.

How I catch it:
- Suspiciously high scores (AUC 0.99 on a hard problem): investigate before celebrating.
- Feature importance: one feature dominating is a red flag.
- Ask for each feature: "When is this value known relative to prediction time?"
- Use pipelines (sklearn `Pipeline`) so preprocessing is fitted inside each fold; use time-based and group-based splits; dedupe across splits.
- Backtest on a true out-of-time period.

Example: a loan default model scored AUC 0.97. `collections_calls_count` was populated only after a default happened. Removing it gave 0.81, which matched production.

**Follow-ups:**
- *How does this apply to RAG evals?* → Don't tune prompts on the test set; keep a held-out eval split.

**Red flags:**
- Standardizing the whole dataset before `train_test_split`.

---

<a id="q9"></a>
## Q9. Cross-validation: when and which kind?
*[Added]*

**What the interviewer is testing:** Matching the validation scheme to the data structure.

**Strong answer:**

Cross-validation estimates generalization more reliably than a single split, especially with small datasets, and gives a variance estimate across folds.

| Scheme | When |
|---|---|
| K-fold (k=5/10) | i.i.d. data, moderate size |
| Stratified K-fold | Classification, especially imbalanced; keeps class ratios per fold |
| Group K-fold | Multiple rows per entity (user, patient, document); avoids group leakage |
| Time-series split / rolling origin | Temporal data; always train on past, validate on future |
| Nested CV | Hyperparameter tuning plus an unbiased performance estimate |
| Holdout only | Very large datasets or expensive models (e.g. LLM fine-tuning) |

Practical rules: never tune on the final test set; report mean ± std across folds; for LLM prompt or eval work, keep a frozen test split and iterate on a dev split.

Example: demand forecasting with daily data. Random K-fold gave MAPE 8%; a rolling-origin split (train through month N, test on N+1) gave 14%, which matched production reality.

**Follow-ups:**
- *Why not CV for LLM fine-tuning?* → Cost. Use a single well-constructed validation set and a separate test set.

**Red flags:**
- Random K-fold on time-series data.

---

## Rapid-fire recap

- Classification needs thresholds; choose them from business cost, not 0.5.
- 99.5% accuracy can mean zero fraud caught; always look at the confusion matrix.
- ROC-AUC looks optimistic on rare positives; PR-AUC shows reality.
- For imbalance: fix metrics, then tune thresholds, class weights, and only then resampling.
- Resample only the training fold.
- In single-label multi-class, micro F1 equals accuracy; macro F1 exposes rare-class failure.
- Always show per-class metrics alongside averages.
- AdamW with warmup and decay is the transformer default; perplexity = exp(loss).
- A big train–val gap means overfitting; both high means underfitting.
- Leakage: ask when each feature is known relative to prediction time.
- Use time-series splits for temporal data and group splits for entities.
