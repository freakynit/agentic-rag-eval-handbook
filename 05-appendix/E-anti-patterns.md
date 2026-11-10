# Appendix E — Fifteen anti-patterns

> Each of these has cost a team at least a quarter. They are listed roughly in order of how often they occur.

---

**1. The aggregate score.**
One number for "quality," reported monthly, moving in the right direction. It cannot tell you what broke, so nobody acts on it. **Instead:** north star plus per-slice table plus failure mix. Always.

**2. Judging what code can check.**
Spending judge tokens on "did it call the right tool" or "is the JSON valid." Slow, expensive, non-deterministic, and occasionally wrong for no benefit. **Instead:** deterministic first. Judges only for semantics.

**3. Recall reported at the wrong k.**
`recall@50` in the deck, `k=8` in production. Flatters everything and predicts nothing. **Instead:** fix k at the production budget, state it in every table header.

**4. Document-level evidence labels.**
Can inflate recall and hides the fact that the right document was retrieved in the wrong section. **Instead:** chunk-level, per hop.

**5. Answer-only labels.**
Without evidence labels you cannot compute recall, path recall, or any conditional. You will spend the year guessing. **Instead:** label evidence; it is the highest-leverage hour in the whole program.

**6. One golden trajectory.**
The agent finds a legitimate shortcut and your suite turns red. The team learns to ignore red. **Instead:** required/forbidden/order constraints plus a step budget.

**7. Faithfulness as a proxy for correctness.**
A system can be 100% faithful to a stale document. Appendix D's case scores 1.0 on support and fails the task. **Instead:** report the support × correctness 2×2.

**8. Abstention as a single number.**
A system that refuses everything scores perfectly on unanswerable questions. **Instead:** false-answer rate and over-rejection rate as a pair, plus a calibration curve.

**9. Synthetic-only eval sets.**
Generated questions can inherit source phrasing, retain unrealistically complete constraints, and remain single-hop even when prompted otherwise. You end up measuring your generator. **Instead:** 50–70% real queries, tag by source, report split metrics.

**10. Tuning on the release set.**
Six months of prompt iteration against the same 150 cases. Scores climb, users do not. **Instead:** frozen / working / holdout, with the frozen set never touched during development.

**11. Comparing systems at different token budgets.**
"GraphRAG beats RAG" when the graph arm got 12k tokens and the vector arm 4k. This confounds architecture with context budget. **Instead:** fix corpus, questions, model, context budget, judge, and run count.

**12. Cost and latency on a dashboard, not in a gate.**
They regress monotonically because nothing blocks. **Instead:** absolute ceilings in `gates.yaml`, alongside quality.

**13. Passing an uncalibrated judge into a release decision.**
No TPR/TNR, no human labels, no κ. You have automated a vibe. **Instead:** 200 labelled items, gate on TPR ≥ 0.85 / TNR ≥ 0.90, re-calibrate quarterly.

**14. Gating on a single run.**
Agents are stochastic. You block on noise, the team overrides the gate, and the gate stops existing. **Instead:** schedule all three runs in advance for strict/core cases; any failure blocks pending triage, and mixed outcomes enter the flip set. Use aggregate rate gates for other stochastic cases. Hard-zero failures block immediately; retries never erase them.

**15. A full suite on every PR.**
Forty-minute CI. Within a month someone adds `--skip-evals`, and then everyone does. **Instead:** 40-case smoke in minutes, full suite nightly, pass^k on 30 cases pre-release.

---

## Three meta-patterns

**Buying the tool before the schema.**
A tracing vendor will show you whatever three fields you emit. Define the spans first (Chapter 4), then choose the UI.

**Letting "model quality" be a bucket.**
It is the bucket that never shrinks because it contains everything. Decompose until each item has an owner and a fix (Chapter 12).

**Building the eval system as the deliverable.**
Immaculate reports about a product nobody improves. The output of the program is **fixed failure buckets**, not dashboards. If the top bucket has not changed in two quarters, the eval system is working and the engineering is not.

---

## The one-line test

For any metric, ask:

> **"If this number got 5 points worse, what would we do on Monday?"**

If the answer is "investigate," it is not a metric yet. It is a measurement looking for a decision.
