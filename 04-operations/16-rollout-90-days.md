# Chapter 16 — Rollout: the first 90 days

> **Chapter 16** · Part IV: Operating it
> **Time:** 20 minutes
> **You will leave with:** a four-phase plan, the roles you need, and a maturity model with exit criteria so you can tell whether it is working.

---

## 16.1 The shape of a rollout

Eval programs fail in two ways: they start with tooling, or they start with everything. Start with **one product surface, one metric, and one gate**, then expand.

The plan below assumes one engineer at 50–70% plus part-time help. With two full-time people, parallelise annotation and instrumentation; shorten the schedule only when the exit criteria are met.

```
Weeks 1–2    Instrument and define
Weeks 3–4    First eval set and first score
Weeks 5–8    Attribution and the graph audit
Weeks 9–12   Gates, cadence, and the loop
```

---

## 16.2 Weeks 1–2 — Instrument and define

**Goal:** you can reconstruct what happened in any run.

Deliverables:

- [ ] Span schema written down (Chapter 4), including `chunk_id`, `resolved_node_id`, `link_score`.
- [ ] Instrumentation in place at the executor boundary, covering retrieval, graph query, LLM call, tool call.
- [ ] `case_id` on eval runs; `trace_id` propagated from the request.
- [ ] Sampling policy and retention policy agreed.
- [ ] The **success definition** written as checkable conditions (Chapter 10), signed off by product.
- [ ] The north star + three guardrails chosen (Chapter 2).

Do **not** start scoring yet. Two weeks of instrumenting feels slow and saves a quarter.

Exit criteria: pick three recent runs at random, and a colleague who was not involved can explain what happened and where it went wrong from the traces alone.

---

## 16.3 Weeks 3–4 — First eval set and first score

**Goal:** a number you trust for one slice.

Deliverables:

- [ ] 60–80 cases, concentrated in **single-hop and multi-hop** (the two slices that carry most traffic and most failures).
- [ ] Gold answers, chunk-level evidence, and (for multi-hop) per-hop evidence.
- [ ] 10 unanswerable cases with `must_abstain`.
- [ ] Retrieval scorer: evidence recall@k, all-evidence recall.
- [ ] Generation scorer: claim support, correctness, completeness.
- [ ] One judge, calibrated on 60 human labels for directional use; do not gate on it until you reach 150–200 (Chapter 13). Yes, this early.
- [ ] A first report, even if ugly.

Exit criteria: you can state, with a denominator, what fraction of multi-hop questions fail because evidence was missed versus because the model misread it. If you cannot, the set or the labels are not ready.

Common trap: spending these two weeks building a beautiful dashboard. Skip it. A markdown report and a spreadsheet are fine for the first month.

---

## 16.4 Weeks 5–8 — Attribution and the graph audit

**Goal:** know which component to fix, every time.

Deliverables:

- [ ] Trajectory scorer: tool selection, argument accuracy, step efficiency, termination, loops.
- [ ] The nine-bucket classifier running on every failed case (Chapter 12).
- [ ] `pass^3` on 30 core cases; the flip set published.
- [ ] If you have a graph: the construction audit (Chapter 7) — 200 entities, 200 relations, 150 resolution pairs, 50 chunks of coverage, 50 summaries.
- [ ] If you have a graph: entity-link accuracy (150 mentions), path recall on 40 multi-hop cases, and the three-way conditional table (Chapter 8).
- [ ] Cost per step and per run; p50/p95 latency (Chapter 11).

Exit criteria: for the last 20 failures, someone other than the system's author can name the bucket without opening a code editor.

This is where the graph audit usually produces an uncomfortable finding. Budget the emotional energy: the most common outcome is that relation recall is far below what the team assumed, and the roadmap changes. That is a success, not a setback — you found it before a customer did.

---

## 16.5 Weeks 9–12 — Gates and cadence

**Goal:** the process runs without you pushing it.

Deliverables:

- [ ] T0 contract tests in CI (30+ assertions, < 30s).
- [ ] T1 smoke set (40–60 cases) blocking PRs, with the 3-run flake policy.
- [ ] T2 nightly full run with per-slice reporting and a paired comparison against `main`.
- [ ] `gates.yaml` with absolute floors, relative drops, per-slice guards, hard zeros.
- [ ] The "what blocks a release" list published.
- [ ] Weekly 30-minute failure review on the calendar, with an owner.
- [ ] Canary process with a pre-registered primary metric.
- [ ] The trace → annotation → regression-case loop working end to end (even if manual).

Exit criteria: a deliberately seeded regression is blocked and a rollback drill succeeds, with the team following the written policy. A real release need not fail to prove the gate works.

---

## 16.6 Roles

Keep it small and explicit. Diffuse ownership is why eval programs stall.

| Role | Time | Responsibility |
|---|---|---|
| **Eval owner** (1 person) | 50% | The set, the scores, the report, the weekly review. Accountable for the numbers. |
| **Instrumentation owner** (1) | 20%, weeks 1–4 | Span schema, emitters, sampling. Hands off after week 8. |
| **Judge owner** (1) | 15% | Calibration, versioning, drift checks, cost. |
| **Graph owner** (if applicable) | 30%, weeks 5–8 | Construction audit, link accuracy, path recall. |
| **Domain labeller** (1–2) | 4–6 h/week | Gold evidence, adjudication, monthly label audit. |
| **Product partner** | 2 h/week | Success definition, severity weighting, abstention policy. |

Two governance rules:

- **The eval owner does not own the features they eval.** Otherwise the numbers become a defence of their own work.
- **The product partner signs the success definition.** Engineers will otherwise optimise something measurable and slightly wrong.

---

## 16.7 Maturity model

Use this to decide what to do next, and to explain progress to leadership without a single composite score.

| Level | Name | You have | Exit criterion |
|---|---|---|---|
| **0** | Blind | Output-only tests, or none | Can replay any run step-by-step |
| **1** | Instrumented | Traces, a 60-case set, retrieval + generation scores | Can attribute failures to a component |
| **2** | Diagnostic | Trajectory scoring, failure taxonomy, conditional reports | 20 consecutive failures correctly bucketed by a non-author |
| **3** | Gated | CI tiers, per-slice gates, weekly review, judge calibration | Seeded regression blocked and rollback drill completed |
| **4** | Self-improving | Online evals, canary, trace→case loop, cost-aware routing | Top failure bucket changes quarter over quarter because you fixed it |

Most teams should aim for level 3 within two quarters and stop optimising the eval system. Level 4 is a steady state, not a project.

A warning about level 4: it is easy to build an eval machine that produces immaculate reports about a product nobody is improving. The output of the program is **fixed failure buckets**, not dashboards. If the top bucket is the same for two quarters, the eval system is working and the engineering is not.

---

## 16.8 Getting buy-in

Three arguments, in order of effectiveness with different audiences.

**For engineering:** "You will stop debugging from screenshots. Every failure names a component in under a minute." Concrete, immediate, and true.

**For product:** "We can tell you which 15% of queries we are bad at, and what fixing them costs." Converts a vague quality complaint into a scoped decision.

**For finance/leadership:** "Cost per successful task is $0.041, and 60% of failures are retrieval misses we can fix without changing models." Two numbers and a lever. Far more persuasive than a faithfulness score.

The argument that does **not** work: "we need evals because best practice." Nobody funds best practice. Fund the ability to answer "why was this answer wrong" and "is the graph worth its cost."

---

## 16.9 The failure modes of the rollout itself

| Symptom | Real cause | Fix |
|---|---|---|
| Eval set keeps growing, never used | No owner, no gate | Cut to 60 cases; wire into PRs this week |
| Team argues about metric definitions every meeting | No written definitions | Appendix A, one page, agreed once |
| Judges disagree with humans | Never calibrated | Chapter 13, 200 labels, two annotators |
| Gate gets overridden under deadline | Thresholds unrealistic or flaky | Loosen absolute floors, tighten the flake policy |
| Reports nobody reads | No decision attached | One page, monthly, top bucket + action |
| Graph audit is "in progress" for a month | Sampling scope too large | 200 entities, 200 relations, 50 chunks. Stop there. |
| Scores improve, users do not | Working-set overfitting | Frozen/holdout split, and check the online proxy |

---

## Do this on your system

- [ ] Pick the single product surface for the first 90 days. Write it down.
- [ ] Assign the eval owner and get 50% of their time, in writing.
- [ ] Put the four phases on a calendar with the exit criteria as milestones.
- [ ] Book the weekly 30-minute review now, with a recurring owner.
- [ ] Assess your current level (0–4) honestly, and pick the next one.
- [ ] Prepare the one-page leadership update template: north star, three guardrails, top bucket, action.
- [ ] Commit to one thing: no incident closes without an eval case.

## Key takeaways

1. Instrument before you score; two weeks of tracing pays for itself many times.
2. Start with 60–80 cases on two slices, not 1,000 across twelve.
3. The graph audit belongs in weeks 5–8, and it will change your roadmap. That is the point.
4. Verify gates and rollback with deliberate drills; do not wait for a production regression.
5. The measure of success is a shrinking top failure bucket, not a growing dashboard.

---

**Next:** the [appendices](../05-appendix/A-metric-reference.md) — reference tables, schemas, judge prompts, and a worked example you can copy.
