# Chapter 11 — Cost, latency, and is the graph worth it

> **Chapter 11** · Part III: System evals
> **Time:** 20 minutes
> **You will leave with:** a TCO model, a cost-adjusted success metric, and an ablation protocol that produces a defensible keep/kill decision.

---

## 11.1 Why this chapter decides whether the project survives

Published cross-paradigm evaluations consistently find the same shape: GraphRAG can beat vector RAG on multi-hop, entity-centric, and global-summarisation questions — and it frequently loses or ties on simple fact lookup, while costing substantially more to build, store, and query. The recurring practical conclusion is not "graphs are good" or "graphs are bad," it is: **graphs pay off on a specific slice, and you must measure whether your slice is that slice.**

Teams that skip this chapter build a system that wins a demo, then dies in the second budget review when someone asks why the index costs 4x and p95 doubled.

---

## 11.2 The TCO model

Four line items. Most teams track only the last one.

### 1. Build cost (one-off + per rebuild)
```
build_cost = extraction_input_tokens × input_price_per_token
           + extraction_output_tokens × output_price_per_token
           + summary_input_tokens × summary_input_price_per_token
           + summary_output_tokens × summary_output_price_per_token
           + embedding_tokens × embedding_price_per_token
           + resolution_compute
           + human_review_hours × rate
           + engineer_hours × rate
```
Also record **build wall-clock time**. A 14-hour rebuild blocks a corpus update; that is a product constraint, not an ops detail.

### 2. Storage cost (monthly)
```
storage_cost_monthly = Σ(store_size_gb × price_per_gb_month)
                     + provisioned_capacity_and_backup_costs
```
Community-summary approaches can be much larger than the graph itself because summaries are generated text stored per community, and re-generated when communities shift. Measure it; do not estimate from node count.

### 3. Query cost
```
query_cost_usd = Σ(tokens_in × input_price_per_token
                 + tokens_out × output_price_per_token)
               + graph_query_compute_usd + vector_search_usd + rerank_usd
```
Attribute per step, per run. The interesting number is the **cost distribution**, not the mean: agents have a long tail, and the p95 run often costs 5–10x the median.

### 4. Operational cost
Rebuild frequency, on-call load, eval compute, annotation hours, vendor minimums. Frequently 20–30% of the total and always omitted.

### The one number to report

```
cost_per_successful_task = total_cost_over_period / successful_tasks_over_period
```

Not cost per query. An agent that fails cheaply by looping is still expensive. If query cost is 3x and success rate only 2x, query cost per success is 1.5x, not lower. Higher spend is justified only when additional successes or other benefits cover it. State whether the numerator is query-only or allocated TCO. Amortise build and rebuild costs over a stated horizon and traffic volume, include failed attempts, and report the ratio as undefined/infinite when there are zero successes.

---

## 11.3 Latency: report the tail

```
p50, p95, p99, and the count of runs > SLO
```

Agents have multimodal latency distributions: a 1-retrieval run at 900ms and a 9-retrieval run at 14s. A p50 hides the second entirely. Report:

- **Latency by path length** (steps 1–3 vs 4–7 vs 8+).
- **Time to first token** if the UI streams.
- **Retrieval latency by type** — graph traversal with LLM-based entity expansion is often the slowest single step; direct community matching is often among the fastest. If graph latency is your problem, the first question is which graph retrieval strategy you are using, not how fast your database is.

---

## 11.4 The ablation protocol

This is the part that must be done properly, because it is the only evidence that will survive scrutiny.

### Rules

1. **Same corpus.** Identical documents, identical chunking where possible.
2. **Same questions.** The full stratified eval set, not a hand-picked subset.
3. **Same model and prompt** for synthesis. Otherwise you are measuring two changes.
4. **Comparable budgets** — fix the token ceiling for synthesis context and state retrieval depth, call count, and total inference spend. A graph node, chunk, and community summary are different units, so equal k alone is not a fair budget. Separate controlled ablations from best-tuned comparisons at matched cost/latency.
5. **Same judge, same version.** Re-judge both arms in the same batch to avoid judge drift.
6. **Same number of runs per case.** Use pass^3 on the core set.
7. **Report per slice.** The aggregate will lie to you.

### The table you produce

Illustrative mean scores across repeated runs, rounded; these are not single-run binary counts. Report run counts and paired intervals in a real report.

| Slice | n | Vector RAG | GraphRAG | Hybrid | Δ (G−V) | Cost/query V | Cost/query G |
|---|---|---|---|---|---|---|---|
| Single-hop | 80 | 0.91 | 0.89 | 0.92 | −2 | $0.004 | $0.021 |
| Multi-hop | 50 | 0.58 | 0.79 | 0.81 | **+21** | $0.006 | $0.028 |
| Global/agg | 30 | 0.49 | 0.71 | 0.72 | **+22** | $0.009 | $0.041 |
| Entity disambig | 20 | 0.62 | 0.84 | 0.85 | **+22** | $0.005 | $0.026 |
| Temporal | 20 | 0.70 | 0.68 | 0.73 | −2 | $0.006 | $0.030 |
| Unanswerable | 20 | 0.77 | 0.80 | 0.82 | +3 | $0.004 | $0.022 |

Here graph-favoured slices contain 100/220 cases (45% of the evaluation sample). That is not a traffic share unless the sample is representative. Estimate traffic weights independently, then test whether routing preserves those gains after router errors and overhead.

### Decision rule

For each disjoint traffic slice i, estimate its share w_i, success S_i,
query cost C_i, and value per successful task V_i. Compare both pure systems
and the routed system:

```
aggregate_success = Σ w_i × S_i
expected_query_cost = Σ w_i × C_i
incremental_value_per_query = Σ w_i × (ΔS_i × V_i − ΔC_i)
```

Subtract allocated build, rebuild, storage, and operating costs over the
same horizon. Check uncertainty, latency, reliability, and policy constraints.
A small high-value slice can justify a graph; a large low-value gain may not.
Numeric cutoffs such as "15% of traffic" or "6x cost" are product choices,
not universal keep/kill rules.

---

## 11.5 The hybrid option to test

A hybrid design can route by evidence need, using a classifier or the agent's planner. Test it against both pure arms:

```
query ──▶ router ──┬─▶ vector RAG         (single-hop, lookup, 60–80% of traffic)
                   ├─▶ graph local        (entity-centric, multi-hop)
                   └─▶ graph global       (aggregation, summarisation)
```

Evaluate the router as its own component:

- **Routing accuracy** against a labelled query-type set. Target > 0.90.
- **Routing cost** — do not spend a 2k-token LLM call deciding between a $0.004 and a $0.02 path.
- **Fallback behaviour** — when the graph path returns nothing on-path, does it fall back to vector? (It should, and that fallback needs testing.)
- **Cost-weighted success** — the number that should beat both pure arms.

Measure the hybrid arm end to end, including misroutes, router inference, retries, and fallbacks; cost savings cannot be inferred from an oracle slice table.

---

## 11.6 Cost of the agent loop itself

Separate from the graph: agents are expensive because they iterate. Measure and cap:

The ranges below are illustrative hypotheses to test, not expected savings.

| Lever | Possible effect | Eval signal |
|---|---|---|
| Max steps | Linear cost cut | premature termination rate must not rise |
| Max retrieval calls | Linear cut | all-evidence recall must not drop > 2 pts |
| Early stopping on sufficient evidence | 20–40% cut | correctness unchanged is the test |
| Smaller model for routing/planning | 30–60% cut on that step | tool selection accuracy |
| Context compression / summarisation | Large token cut | claim support rate |
| Caching (semantic + exact) | Large cut on repeat traffic | staleness rate |

Every one of these is an eval trade, not an ops setting. Never ship one without the paired metric.

The single most reliable saving in agentic RAG is **stopping earlier**. First check which late steps are redundant. Verification and synthesis may add no evidence while still being necessary. Validate stopping changes against task success and premature-termination rate; savings are not automatically quality-neutral.

---

## Do this on your system

- [ ] Instrument cost per step, per run, and publish the p95, not the mean.
- [ ] Compute `cost_per_successful_task` for your current system. Put it on the monthly page.
- [ ] Run the ablation with all seven rules. Produce the slice table.
- [ ] Apply the decision rule explicitly and write down the outcome.
- [ ] If you keep the graph, build the router and evaluate routing accuracy.
- [ ] Measure `wasted_steps` and test an early-stopping change with the paired quality metric.
- [ ] Record build time and rebuild time. These are product constraints.

## Common mistakes

- **Comparing at different token budgets.** The most common invalid ablation. Fix context size, then compare.
- **Aggregate-only comparison.** Hides the fact that the win is concentrated in 12% of traffic.
- **Cost per query.** Rewards cheap failure.
- **Ignoring rebuild cost.** Year-two rebuilds are where GraphRAG budgets break.
- **Reporting p50 latency.** Agents are tail-latency machines.
- **Shipping a cost optimisation without its paired quality metric.** You will trade one regression for another and not notice for a quarter.

## Key takeaways

1. Four cost lines: build, storage, query, operational. Most teams track only query.
2. Report cost per *successful* task and p95 latency, always.
3. Ablations are only valid when corpus, questions, model, budget, judge, and run count all match.
4. Decide per slice; test the routed system including router errors and overhead.
5. The cheapest large win is usually better stopping criteria, not a smaller model.

---

**Next:** [Chapter 12 — Failure taxonomy](12-failure-taxonomy.md). Turning a wall of red numbers into a ranked list of fixes.
