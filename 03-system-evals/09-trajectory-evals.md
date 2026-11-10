# Chapter 9 — Trajectory evals

> **Chapter 9** · Part III: System evals
> **Time:** 30 minutes
> **You will leave with:** eight process metrics, a way to define acceptable paths without over-constraining, and the pass^k reliability procedure.

---

## 9.1 What a trajectory is, precisely

A trajectory is the ordered sequence of decisions and actions in one run:

```
step 0: plan      → decompose into 2 sub-queries
step 1: retrieve  → sq1 "suppliers list"
step 2: retrieve  → sq2 "2023 audit"
step 3: graph     → link "Northwind" → expand 2 hops
step 4: reason    → merge evidence, notice one supplier missing
step 5: retrieve  → sq3 "Calder Freight contract status"   ← replan
step 6: synthesize→ final answer
```

Everything in this chapter is a function of that sequence. If your traces look like `(question, answer)`, go back to Chapter 4.

---

## 9.2 The eight process metrics

### 1. Tool selection accuracy
> Did the agent choose the right tool for each step?

Deterministic. Compare `tool_name` at each step to the acceptable tool set for that step in the reference path. Score per step, then per run (`all steps correct`).

This catches the most basic failure: using vector search when the question needs graph traversal, or using the graph for a lookup that a keyword filter would answer.

### 2. Argument correctness
> Were the tool's inputs right?

Deterministic where possible: entity names, IDs, date ranges, filters, `top_k`, hop depth.

This is where subtle bugs live. `graph_neighbors(entity="Northwind Logistics")` with the full legal name may miss because your nodes are keyed on short names. `search_docs(date_after="2023-01-01")` may exclude the document the user meant. Argument errors are high-frequency and cheap to detect.

Split into:
- **Present and valid** (does it satisfy the tool schema) — deterministic.
- **Semantically right** (does it name the right entity/range) — deterministic against the case's expected values, or judge for fuzzy cases.

### 3. Step efficiency
> Did it take the shortest reasonable path?

```
step_efficiency = reference_steps / actual_steps             (capped at 1.0, higher is better)
```

Only meaningful for successful runs with a positive reference count. Failed runs receive `None`, so a premature one-step answer cannot earn perfect efficiency. Use the same definition for reference and actual steps. Do not compute it on cases with many legitimate paths; it will punish valid exploration.

Report the distribution, not the mean. Means hide the tail, and the tail is where your cost lives.

### 4. Path match
> Did the trajectory contain the required steps, avoid the forbidden ones, and stay in order?

Three variants, in increasing strictness:

| Variant | Check | Use when |
|---|---|---|
| **Set match** | All required tools present, no forbidden tools | Many valid orderings |
| **In-order** | Required steps appear in relative order | Order matters (link before expand) |
| **Exact** | Matches the reference sequence step-for-step | Rare; brittle; avoid for gating |

Most teams should gate on set match and monitor in-order. Exact match produces false regressions every time the agent finds a legitimate shortcut.

### 5. Loop and thrash detection
> Did the agent repeat itself without progress?

Deterministic detectors, run on every trace:

- **Identical repeat** — same tool + same normalised arguments ≥2 times.
- **Oscillation** — A→B→A→B pattern.
- **No-progress window** — last 3 steps added no new evidence (compare retrieved chunk IDs / node IDs to the accumulated set).
- **Query rewording loop** — same query with only stopword changes.

Report `loop_rate` (% of runs with ≥1 detector firing) and `no_new_evidence_steps`. Treat these detectors as flags: retries, verification, reasoning, and final synthesis may be necessary without adding evidence. Count `wasted_steps` only after checking the step purpose; do not label every step after the last retrieval as waste.

### 6. Termination correctness
> Did it stop at the right time, for the right reason?

| Termination | Expected | Failure |
|---|---|---|
| `final_answer` after sufficient evidence | ✅ | |
| `final_answer` with insufficient evidence | ❌ **premature** (under-retrieval) | |
| `max_steps` reached | ⚠️ usually a planning failure | |
| `abstain` when unanswerable | ✅ | |
| `abstain` when answerable | ❌ over-rejection | |
| `error` / `timeout` | ❌ | |

Score `premature_rate` and `budget_exhaustion_rate` separately. They point at opposite fixes: better stopping criteria vs. better planning.

### 7. Replanning quality
> When evidence contradicted the plan, did the agent adapt?

Judge-scored, using the trace. Criteria:

- Did it notice the contradiction (e.g. the document says the supplier was *not* reviewed)?
- Did it change course, or push the original plan through?
- Was the new plan reasonable, or a random walk?

This is hard to score at scale. Sample 30 runs where a contradiction exists (you can construct these deliberately) and score them.

### 8. Error recovery
> When a tool failed, did it recover sensibly?

Inject failures deterministically in a test harness: timeout, empty result, schema error, 500. Then check:

- Retry with backoff vs. blind retry vs. immediate give-up.
- Fallback to an alternative tool vs. dead end.
- Whether the final answer acknowledges the gap.

Target: recovery rate > 0.8 on transient failures, and **zero** silent failures (answering as if the empty result were the truth).

---

## 9.3 Defining acceptable paths without over-constraining

The fastest way to make trajectory evals useless is to encode one "correct" path for every question. Agents are allowed to be clever; your eval should reward that.

Use a **constraint spec**, not a script:

```yaml
path_constraints:
  must_include:
    - {tool: "search_docs"}
    - {tool: "graph_neighbors"}
  must_not_include:
    - {tool: "web_search"}          # out-of-corpus is never allowed here
    - {tool: "send_email"}          # side effects forbidden in this eval
  order_pairs:
    - ["search_docs", "graph_neighbors"]   # relative order only
  max_steps: 10
  max_retrieval_calls: 5
  must_terminate_with: "final_answer"
```

Validate this mechanically for a fixture whose contract requires both tools. For outcome-only cases, allow any route that obtains sufficient valid evidence; a valid vector-only shortcut should not fail a graph-tool requirement.

For a small number of core cases (20–30), also record a **reference step count** (`reference_steps`), used only for step-efficiency scoring. Never gate on exact match.

---

## 9.4 pass^k: reliability as a first-class metric

Run the same case k times (typically k=3 on a core set, optionally k=5 before release) and require **all** runs to succeed. Schedule all k attempts in advance, including on cases whose first attempt passes. Reset session state and side-effect fixtures between attempts; pin corpus, graph, and configuration, and record cache policy.

```
pass^k = fraction of cases where all k runs succeeded
```

Why this matters more than pass@1:

- An agent with pass@1 ≈ 0.85 and pass^3 ≈ 0.61 is reliable on only 61% of cases across three runs; 39% have at least one failure. Users do not see your average; they see one draw.
- pass^k exposes nondeterminism from sampling, retrieval variance, and race conditions in tool calls.
- It makes flaky cases visible. A case that passes 2 of 3 runs is a case with a latent bug, not noise.

Report the **flip set**: cases that pass sometimes. That list is your highest-value debugging queue, because each one has a reachable correct path — the system knows how to succeed and does not reliably choose it.

For a task with success probability p under independent, identically configured runs, all-three success is p^3: 0.85^3 ≈ 0.61. Across tasks, report mean(p_i^k), not mean(p_i)^k. The observed fraction of all-pass groups estimates this rate and needs an interval across cases; 30 cases gives a coarse estimate.

If n ≥ k runs are available per case with c successes, use `comb(c, k) / comb(n, k)` and average across cases, as in [τ-bench](https://arxiv.org/abs/2406.12045). For n=k this is exactly the all-pass indicator.

Cost note: pass^3 triples eval cost. Start on a 30-case core set, then expand if a gate needs finer precision.

---

## 9.5 Multi-turn state

For conversational agents, add these to the trajectory scorer:

| Check | Method |
|---|---|
| Constraint retention | Turn 4 answer still respects a filter stated in turn 2 (deterministic: check the filter appears in the applied query) |
| Unnecessary re-fetch | Avoid repeated retrieval unless a freshness, verification, or permission check requires it |
| Reference resolution | "And by region?" resolves to the right prior entity |
| Correction handling | User says "no, the other John Smith" → does the agent relink and re-expand? |
| Drift | Answer to turn 5 is not contaminated by an abandoned branch from turn 3 |

Multi-turn trajectories are scored as a whole run with sub-spans per turn. Do not average turn scores; the failure is relational.

---

## 9.6 A scoring sketch

```python
def score_trajectory(trace, case):
    # Adapter: spans expose defaults for optional attributes (Appendix B).
    constraints = case.path_constraints
    step_ids = {s.step_index for s in trace.spans if s.step_index is not None}
    # One logical action per attempted call, including validation failures.
    actions = sorted((s for s in trace.spans if s.tool_name),
                     key=lambda s: (s.step_index, s.started_at, s.span_id))
    tool_seq = [s.tool_name for s in actions]
    violations = []

    def matches(span, spec):
        return (span.tool_name == spec["tool"] and
                all(span.arguments.get(key) == value
                    for key, value in spec.get("args_contains", {}).items()))

    for req in constraints.get("must_include", []):
        if not any(matches(s, req) for s in actions):
            violations.append(f"missing required call: {req}")
    for ban in constraints.get("must_not_include", []):
        if any(matches(s, ban) for s in actions):
            violations.append(f"forbidden call: {ban}")

    # Tool-name order pairs mean at least one occurrence of A precedes B.
    # For parallel calls needing completion-before-start, compare timestamps.
    for a, b in constraints.get("order_pairs", []):
        if not any(x == a and y == b for i, x in enumerate(tool_seq)
                   for y in tool_seq[i + 1:]):
            violations.append(f"missing ordered pair: {a}->{b}")

    retrieval_calls = sum(s.span_type in {"retrieval", "graph_query"} for s in actions)
    if len(step_ids) > constraints.get("max_steps", float("inf")):
        violations.append("step budget exceeded")
    if retrieval_calls > constraints.get("max_retrieval_calls", float("inf")):
        violations.append("retrieval budget exceeded")
    expected_end = constraints.get("must_terminate_with")
    if expected_end and trace.outcome.terminated_reason != expected_end:
        violations.append("wrong termination reason")

    arg_accuracy = (sum(s.schema_valid and case.args_match(s) for s in actions)
                    / len(actions)) if actions else None
    eff = (min(1.0, case.reference_steps / len(step_ids))
           if case.success(trace) and case.reference_steps
           and case.reference_steps > 0 and step_ids else None)
    seen, repeats = set(), 0
    for s in actions:
        key = (s.tool_name, normalise(s.arguments))  # hashable canonical JSON
        repeats += key in seen
        seen.add(key)

    return {
        "path_ok": not violations,
        "violations": violations,
        "arg_accuracy": arg_accuracy,
        "step_efficiency": eff,
        "repeat_flags": repeats,
        "terminated": trace.outcome.terminated_reason,
        "steps": len(step_ids),
        "retrieval_calls": retrieval_calls,
    }
```

`path_ok` checks the declared structural constraints; it does not prove that evidence was sufficient. The product success scorer must also check required argument semantics, evidence sufficiency, and policy. For task pass^k, apply that same success definition to every scheduled run, then take the all-k conjunction. Report trajectory compliance separately unless it is an explicit task requirement.

---

## 9.7 Targets

| Metric | Unacceptable | Acceptable | Good |
|---|---|---|---|
| Tool selection accuracy | < 0.85 | 0.92–0.97 | > 0.99 |
| Argument accuracy | < 0.80 | 0.90–0.96 | > 0.98 |
| Path compliance (constraints) | < 0.80 | 0.90–0.95 | > 0.97 |
| Loop rate | > 0.20 | 0.05–0.12 | < 0.03 |
| Premature termination | > 0.15 | 0.05–0.10 | < 0.03 |
| Budget exhaustion | > 0.10 | 0.02–0.06 | < 0.02 |
| step_efficiency (median) | < 0.5 | 0.65–0.80 | > 0.85 |
| pass^3 (core set) | < 0.65 | 0.75–0.88 | > 0.92 |
| Failure recovery | < 0.5 | 0.7–0.85 | > 0.9 |

---

## Do this on your system

- [ ] Write `path_constraints` for 40 cases (must/must-not/order/max_steps).
- [ ] Score tool selection and argument correctness deterministically. Both should be near 1.0 before you worry about anything else.
- [ ] Implement the four loop detectors and report `wasted_steps`.
- [ ] Classify every termination reason; compute premature and budget-exhaustion rates.
- [ ] Run pass^3 on 30 core cases and publish the flip set.
- [ ] Inject tool failures (timeout, empty, schema error) and score recovery.
- [ ] For multi-turn cases, add the five state checks.

## Common mistakes

- **One golden trajectory.** You will fight false regressions forever.
- **Gating on exact path match.** Punishes legitimate shortcuts.
- **Averaging turn scores in multi-turn.** The failure is relational, not per-turn.
- **Ignoring argument correctness.** Most "the graph is bad" complaints are actually `entity="Northwind Logistics Ltd"` not matching `supplier:northwind`.
- **Reporting pass@1 for an agent.** It is the least predictive reliability number you can publish.
- **Not scoring termination.** Over-searching is the most common hidden cost.

## Key takeaways

1. Eight process metrics: tool choice, arguments, efficiency, path compliance, loops, termination, replanning, recovery.
2. Constrain paths, do not script them. Deterministic constraint checks are fast and stable.
3. pass^k, not pass@1. The flip set is your best debugging queue.
4. Argument correctness is high-frequency and cheap to measure; check it early.
5. Premature termination and budget exhaustion point at opposite fixes; never combine them.

---

**Next:** [Chapter 10 — End-to-end, reliability, and online evals](10-end-to-end-and-online.md). The number the business sees.
