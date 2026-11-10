# Chapter 13 — LLM-as-judge operations

> **Chapter 13** · Part IV: Operating it
> **Time:** 25 minutes
> **You will leave with:** a rubric design pattern, a calibration protocol with real acceptance thresholds, and a bias-control checklist.

---

## 13.1 What judges are for, and what they are not

A judge is a **measurement instrument**, not an oracle. Treat it like a lab assay: it has a sensitivity, a specificity, a calibration date, and an expiry.

Use a judge for criteria that are semantic and otherwise unmeasurable:

- Is this claim supported by this chunk?
- Is this trajectory a reasonable approach to the question?
- Does this summary represent the community?
- Did the agent recover sensibly from the tool failure?

Do **not** use a judge for: tool names, IDs, JSON validity, counts, dates, set membership, latency, token counts. These are deterministic and a judge will occasionally get them wrong for no benefit.

Rule of thumb: if you can describe the check to a junior engineer in one sentence with no examples, it is not a judge task.

---

## 13.2 Rubric design

### Binary criteria, not scales

Replace "rate faithfulness 1–10" with a set of yes/no questions.

```yaml
criteria:
  - id: c1
    question: "Is every numeric value in the answer stated in the context?"
    verdicts: ["yes", "no"]
  - id: c2
    question: "Does the answer avoid naming an entity that does not appear in the context?"
    verdicts: ["yes", "no"]        # yes = good; keep every criterion oriented this way
```

Binary verdicts give you: stable inter-rater agreement, a meaningful aggregate (a proportion), and easy failure inspection. Ten-point scales give you: judge drift, range compression, and arguments about whether something is a 6 or a 7. Mixed-polarity criteria invite sign errors when you aggregate; phrase every criterion so "yes" means "good."

If you need tri-state (supported / partial / unsupported), use it — but define "partial" with an example in the rubric, or judges will use it as a shrug.

### One criterion per judge call

Do not ask a judge to evaluate five things at once. Multi-criteria prompts produce correlated scores and make failures unattributable. Call the judge five times, or use a structured schema with independent fields and score them independently.

### Evidence-conditioned judging

The single biggest quality improvement available: **give the judge the evidence, not just the answer.**

```
Bad:  "Is this answer correct?"  (judge must guess the truth)
Good: "Here are the source chunks and the required facts.
       For each required fact, is it present in the answer?
       Is each answer claim supported by a chunk?"
```

Unconditioned judges default to plausibility, which correlates with fluency, which correlates with confident hallucination. This is how you get a judge that scores a wrong answer highly.

For absolute correctness, score against required facts and reference evidence. Use **pairwise comparison** to measure preference or relative improvement, with both candidate orders and a tie option. Preference over a previous version does not establish correctness: both answers may be wrong, or both may be valid. Position and verbosity biases still require checks.

### Force a justification, then discard it

Ask for a short justification before the verdict. It improves accuracy. Do not parse the justification into your metric; it invites judge-shopping and is not reliable at scale. Log it for humans debugging disagreements.

---

## 13.3 The biases, and what to do

| Bias | Symptom | Mitigation |
|---|---|---|
| **Position** | In pairwise, the first (or last) candidate wins | Randomise order; run both orders and require agreement, else mark ambiguous |
| **Verbosity** | Longer answers score higher | Length-normalise; add an explicit "ignore length and style" instruction; test with a padded-but-empty answer |
| **Self-preference** | Judge favours output from its own model family | Use a different family than the generator; validate |
| **Sycophancy / agreement** | Judge agrees with a stated "expected answer" even when wrong | Do not show the expected verdict; show evidence only |
| **Leniency drift** | Scores creep up over weeks | Pin the judge model version; re-run a fixed calibration set monthly |
| **Format sensitivity** | Markdown vs plain changes the score | Normalise formatting before judging |
| **Rubric ambiguity** | Low inter-annotator agreement | Inspect class balance, rubric clarity, and disagreements |

Position bias is not theoretical. Cross-paradigm evaluations of summarisation have shown that simply reversing the order of two candidate summaries can flip the verdict — enough to change which system "wins." If you compare two systems pairwise, always run both orders.

---

## 13.4 Calibration protocol

You need 150–200 human-labelled items before you can trust a judge. Do this once, then quarterly and after any judge model or prompt change.

### Procedure

1. Sample 200 items stratified across slices and difficulty, including at least 40 hard/negative cases.
2. Two humans label independently; adjudicate disagreements (this also measures rubric clarity).
3. Run the judge on the same 200.
4. Build the confusion matrix against adjudicated human labels. Reserve a held-out calibration subset if you tune the judge prompt on these items.
5. Compute **agreement**, **TPR** (judge catches real failures), **TNR** (judge does not invent failures), and **Cohen's κ**.

```
                 human: fail   human: pass
judge: fail           TP            FP
judge: pass           FN            TN

TPR (sensitivity) = TP / (TP + FN)     ← does it catch bad answers?
TNR (specificity) = TN / (TN + FP)     ← does it leave good answers alone?
```

### Acceptance thresholds

| Use | Min TPR | Min TNR | Min κ |
|---|---|---|---|
| Blocking a release (gate) | 0.85 | 0.90 | 0.65 |
| Dashboard / trend monitoring | 0.75 | 0.80 | 0.55 |
| Exploratory analysis | 0.65 | 0.70 | — |

Asymmetric thresholds are deliberate. For a release gate, a false accusation (FP) wastes engineering time and erodes trust, while a miss (FN) ships a bug. Tune to your cost ratio, and write the ratio down.

**Report sampling uncertainty and judge measurement error separately.** Here failure is the positive class: TNR 0.90 means 10% of human-pass items are falsely marked fail, which depresses the observed support rate. Missed failures (1−TPR) inflate it. Their net effect depends on failure prevalence. A naive bootstrap of judge outputs does not correct systematic judge bias; use human audits or a validated correction method such as [prediction-powered inference in ARES](https://arxiv.org/abs/2311.09476). Report class denominators and intervals for TPR/TNR; 200 total labels may still leave very few failures in an individual slice.

### Judge the judge

Keep the 200-item calibration set as a **regression test for the judge itself**. When you upgrade the judge model or edit its prompt, re-run it. A judge change that silently shifts every metric by 5 points is the most confusing incident you will have this year. Pin the version, and put the judge's own pass rate on the dashboard.

---

## 13.5 Cost control

Judges are the largest line item in most eval budgets. Five levers, in order of payoff:

1. **Cascade.** Cheap deterministic checks first; only unresolved items go to the judge. In practice 40–60% of cases are decidable deterministically (empty answers, missing citations, wrong tool).
2. **Cache by content hash.** Key by answer, context, rubric/prompt version, judge model/version, and inference settings. Include tool/reference inputs where relevant. Repeats intended to measure judge variability must bypass this cache. Eval re-runs hit cache constantly.
3. **Sample.** Nightly full judge on 150 cases; PR smoke judged on 40. You do not need a judge on all 5,000 synthetic cases.
4. **Smaller judge where calibrated.** If a small model reaches TPR/TNR thresholds on *your* task, use it for monitoring and keep the strong judge for gates. Validate separately per criterion; semantic faithfulness may need a stronger judge, while labelled tool-choice checks usually need no model.
5. **Batch.** Most providers price batch inference far below interactive. Nightly evals are a perfect fit.

Track `judge_cost_per_case` and `judge_calls_per_run` on the same dashboard as system cost. Eval spend that nobody watches grows 10x quietly.

---

## 13.6 Data boundary

Before you send anything to a hosted judge, answer these in writing:

- Does the judge see customer documents? If yes, is that permitted by contract and policy?
- Which region does the judge process in? Cross-border transfer rules apply to eval data too.
- Is the judge provider's retention/training policy acceptable for your data classification?
- Do you need a self-hosted judge? Choose hosting and model size from your data constraints and measured calibration results; parameter count alone does not establish adequacy.

The pragmatic pattern in regulated enterprises: **self-hosted judge for anything touching customer content, hosted judge for synthetic and public data.** Calibrate both; they will disagree slightly, so do not mix them within one time series.

---

## 13.7 Judge prompt hygiene

- **Version the prompt** like code (`faithfulness_v7`), and store the version on every score.
- **Never edit a judge prompt and a system prompt in the same release.** You will not know which moved the number.
- **Freeze the rubric text** in the eval repo; do not let it live only in a dashboard UI.
- **Include 2–3 worked examples** in the prompt, including one negative and one borderline. Examples do more for agreement than instructions.
- **Constrain output** to a schema (`{"verdict": "yes"|"no", "reason": "..."}`) and validate it. Unparseable output is a silent failure; log `parse_error_rate` and alert above 1%.

Appendix C has four templates to adapt, validate, and calibrate before production use.

---

## Do this on your system

- [ ] List every judged metric you have. For each, state the criterion in one sentence and whether code could check it.
- [ ] Convert the biggest judged metric to binary criteria with evidence conditioning.
- [ ] Build the 200-item calibration set with two annotators and adjudication.
- [ ] Compute TPR, TNR, κ. Compare to the thresholds above. Fix or downgrade the metric.
- [ ] Add the calibration set to CI as a test for the judge itself.
- [ ] Implement content-hash caching and a deterministic-first cascade.
- [ ] Write down the data-boundary decision and get it approved.

## Common mistakes

- **1–10 scales.** They look precise and are not.
- **Judging without evidence.** Produces a fluency detector, not a faithfulness metric.
- **One judge call for five criteria.** Correlated, unattributable scores.
- **Unpinned judge version.** Your metrics will drift with a vendor upgrade you did not make.
- **No FP/FN analysis.** A judge with TPR 0.6 will happily tell you everything is fine.
- **Mixing self-hosted and hosted judges in one time series.** The step change will look like a product regression.
- **Reporting point estimates without denominators and uncertainty.** Decimal places alone do not establish measurement precision.

## Key takeaways

1. Judges are calibrated instruments with sensitivity, specificity, and an expiry date.
2. Binary, evidence-conditioned, one criterion per call.
3. Calibrate against 200 human labels; gate on TPR ≥ 0.85 and TNR ≥ 0.90.
4. Keep the calibration set as a regression test for the judge itself.
5. Cascade, cache, sample, batch. Eval cost is a budget line, not a rounding error.

---

**Next:** [Chapter 14 — CI gates and release process](14-ci-gates-and-release.md). Turning measurement into a decision.
