# Chapter 10 — End-to-end, reliability, and online evals

> **Chapter 10** · Part III: System evals
> **Time:** 25 minutes
> **You will leave with:** a task-success definition that is not "the answer looked good," a reliability procedure, and an online evaluation design that survives real traffic.

---

## 10.1 Task success is a product definition

"Did the agent answer correctly?" is a component question. Task success asks: **did the user's job get done?**

These diverge all the time. A correct answer with a broken citation, delivered after 40 seconds, in the wrong language, is a failed task.

Define success **per product surface**, and write it down:

| Product | Success means | Not success |
|---|---|---|
| Support assistant | Question resolved, correct policy cited, no escalation needed | Correct but uncited; correct but suggests contacting support |
| Research copilot | All required facts present, sources linked, no contradictions | Correct summary with no traceable sources |
| Internal ops agent | Action performed correctly, side effects exactly as intended, audited | Action performed but not logged |
| Analytics Q&A | Correct number, correct units and period, method stated | Correct number, wrong period |

Then decompose into a **checklist against the trace and the answer**, not a vibe. A machine-checkable success definition:

```
success = correctness_ok
        ∧ completeness_ok
        ∧ citations_ok
        ∧ format_ok
        ∧ latency_ok
        ∧ policy_ok
```

Each conjunct is scored separately elsewhere in this handbook. Task success is their conjunction (or a weighted rule you state explicitly). The value is that a failure now names the conjunct that broke.

### Partial credit, when binary is too blunt

For long multi-part tasks, score progress:

```
progress = weighted_sum(completed_subgoals) / total_weight
```

Report both. Binary success drives release decisions; progress tracks week-over-week improvement on hard slices, where binary success sits at 0 for months and demoralises everyone.

---

## 10.2 Reliability: the number that predicts user experience

Users experience one draw. Your mean describes many. Close the gap with three measurements.

### 1. pass^k
Covered in Chapter 9. Run the core set k=3–5 times, require all to pass.

### 2. Variance decomposition
For 50 cases × 5 runs, split variance into:

- **Case difficulty variance** — differences between tasks; these may improve with better coverage or capabilities.
- **Run-to-run variance within a case** — this is your nondeterminism, and it is a bug budget.

If within-case variance is high, look at: sampling temperature, retriever score ties, graph traversal ordering (set iteration order!), tool retry races, and prompt-assembly order. A surprising amount of agent nondeterminism is an un-ordered `set` in a traversal function.

### 3. Flake accounting
Any case that passes some runs and fails others is **flaky**, and flaky is not noise — it means the system can reach success but does not reliably choose it. Track:

```
flake_rate = cases_with_mixed_outcomes / total_cases
```

Target < 5%. Every flaky case gets an owner and a ticket. The flip set from Chapter 9 is the input.

---

## 10.3 Human evaluation, done cheaply

You cannot judge everything with a model you have not calibrated. You also cannot afford to judge everything with humans. The workable split:

| Purpose | Volume | Method |
|---|---|---|
| Judge calibration | 150–200 items, once + quarterly | Two annotators, adjudication on disagreement |
| New-slice exploration | 50 items per new slice | One annotator, rubric |
| Incident triage | every incident | One expert, freeform + failure label |
| Ongoing spot check | 30/week, random from prod | One annotator, binary success + failure label |

Design notes that make human eval usable:

- **Binary or 3-point rubrics.** "Rate 1–10" produces noise and arguments.
- **Show the trace, not just the answer.** Annotators who cannot see retrieval will label retrieval failures as hallucinations.
- **Show the gold evidence** where it exists, and let them flag bad gold labels. Your labels rot too.
- **Measure inter-annotator agreement** (Cohen's κ). Low κ can reflect rubric ambiguity, label imbalance, or annotator disagreement. Inspect class frequencies, raw agreement, and disagreements before diagnosing it.
- **Timebox**: 2 minutes per item. If it takes 10, the rubric is too complex.

---

## 10.4 Online evaluation

Offline evals predict; online evals confirm. Once you have traffic, run four things.

### 1. Cheap automatic scoring on sampled traffic
Use a uniform sample of production traces for rate estimates (separate from oversampled errors), and run the two or three cheapest high-signal scorers: claim support, citation validity, and a deterministic trajectory check. Do not run the full suite; it is too expensive and too slow.

### 2. Implicit signals
These are free and often more honest than thumbs:

| Signal | Reads as | Caveat |
|---|---|---|
| Re-ask / rephrase within 60s | Failure | Also happens when users refine |
| Abandonment after answer | Weak failure | Could be success (they left satisfied) |
| Citation click then immediate back | Citation was wrong | Strong signal, low volume |
| Escalation to human | Failure (or correct routing) | Segment carefully |
| Copy / export of answer | Success | |
| Thumbs down + comment | Failure, with a reason | Sparse, biased |

Combine signals into a proxy success rate, and **validate it against human labels** quarterly. A proxy nobody validated is a rumour with a dashboard.

### 3. Guardrail and safety metrics
Tracked continuously, alerted on, not just evaluated:

- Prompt-injection success rate (canary documents planted in the corpus).
- PII leakage rate in answers.
- Unauthorized-document retrieval rate (ACL/permission violations) — for enterprise systems this is the metric that gets people fired, and it belongs in the eval gate, not just monitoring.
- Refusal rate on benign queries (over-rejection in the wild).

### 4. Canary and A/B
Promote changes with a small traffic slice, compare on **task success proxy + cost + latency**, and pre-register the metric and the decision rule. Post-hoc metric shopping is how teams ship regressions with a straight face.

Minimum viable A/B design:
- One primary metric, declared before launch.
- Sample size from a power calculation. As a rough guide, at two-sided α=0.05 and 80% power, 300 independent tasks per arm detects roughly a 10–12 point absolute effect; a 5-point effect near a 50–75% baseline needs roughly 1,100–1,600 per arm. These are normal-approximation planning values. Account for clustering by user or tenant, repeated looks, and any within-subject correlation in the actual design.
- Guardrails: p95 latency, cost/success, guardrail violation rate.
- Automatic rollback on guardrail breach.
- Randomise at the user/session level when requests share state; use a fixed horizon or a predeclared sequential method instead of repeatedly checking ordinary p-values.

---

## 10.5 The offline/online gap

Expect offline and online to disagree. Diagnose with this table:

| Offline | Online | Likely cause |
|---|---|---|
| High | Low | Eval set is too easy / too synthetic / overfit; or prod distribution differs (tenants, languages, doc types) |
| Low | High | Eval set is harder than prod (fine, but do not panic); or prod users ask easier questions |
| High | High | Working |
| Low | Low | Real problem; go to Chapter 12 |

The most common is high/low, and the most common cause is that the team spent six months tuning on the working set. Chapter 3's frozen/holdout split is the mitigation.

---

## 10.6 Reporting cadence

| Cadence | Audience | Content |
|---|---|---|
| Per PR | authors | smoke (40 cases), pass/fail |
| Nightly | team | full set, per-slice, failure mix, cost |
| Weekly | eng + PM | trends, flip set, new cases added, incidents |
| Monthly | leadership | task success, cost/success, reliability, top 3 investments |
| Per release | all | gate report, canary result, rollback decision |

Keep the monthly view to one page: north star, three guardrails, top failure bucket, and what you are doing about it.

---

## Do this on your system

- [ ] Write the success definition for your main product surface as a conjunction of checkable conditions.
- [ ] Implement it as a scorer; report the conjunct that fails, not just a boolean.
- [ ] Run 50 cases × 5 times; compute within-case variance and flake rate.
- [ ] Build the flip set and assign owners.
- [ ] Pick three implicit signals and compute a proxy success rate.
- [ ] Validate the proxy against 100 human labels.
- [ ] Add prompt-injection and permission-violation canaries to the eval set.
- [ ] Pre-register the primary metric for your next A/B.

## Common mistakes

- **"The answer looked good" as task success.** Not checkable, not comparable, not actionable.
- **Reporting only the mean.** It describes a user who does not exist.
- **Treating flaky cases as noise.** They are the most fixable bugs you have.
- **Trusting thumbs-up rates.** Sparse, biased toward power users, and confounded by politeness.
- **Shipping on canary without a pre-registered metric.** You will find a metric that improved.
- **Skipping the permission check.** In enterprise RAG, an ACL leak is not a quality regression, it is an incident.

## Key takeaways

1. Task success is a product definition and a conjunction of checkable conditions.
2. Report progress alongside success on hard slices, or the team burns out.
3. Reliability = pass^k + variance decomposition + flake accounting.
4. Online evals confirm; implicit signals are free but must be validated against humans.
5. Pre-register metrics and guardrails before every canary.

---

**Next:** [Chapter 11 — Cost, latency, and is the graph worth it](11-cost-and-graph-ablation.md). The chapter that decides whether any of this survives the budget review.
