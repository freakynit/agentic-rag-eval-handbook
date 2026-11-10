# Chapter 12 — Failure taxonomy and error attribution

> **Chapter 12** · Part III: System evals
> **Time:** 20 minutes
> **You will leave with:** nine primary failure buckets with detectors, an annotation schema, and a weekly review ritual that turns them into a roadmap.

---

## 12.1 Why a taxonomy beats a scoreboard

A scoreboard says "success dropped 4 points." A taxonomy says "eleven entity-linking failures appeared after the alias table changed." Only one of those produces a ticket.

The taxonomy is also the only artifact that lets a newcomer debug the system without reading every trace. It is institutional memory.

---

## 12.2 The nine primary buckets

Assign exactly one **primary** bucket per failed run. Secondary labels are allowed and encouraged, but forcing a primary label is what makes the histogram useful.

| # | Bucket | Definition | Evidence / candidate detector | Owner |
|---|---|---|---|---|
| 1 | **Corpus gap** | The answer does not exist in the corpus, or exists only in a stale/superseded document | Gold evidence search returns nothing; or evidence found only in superseded docs | Data / content ops |
| 2 | **Retrieval miss** | Evidence exists, retriever did not return all of it | `evidence_recall@k < 1` for the relevant sub-query | Retrieval eng |
| 3 | **Graph construction gap** | The path exists in documents but not in the graph | Gold path absent from graph | Graph eng |
| 4 | **Graph reach miss** | The path is in the graph, not fully in the retrieved subgraph | Path in graph ∧ (`node_recall < 1` ∨ `edge_recall < 1`) | Graph / retrieval eng |
| 5 | **Entity link miss** | A named entity that should have been linked is resolved to the wrong node or to null (not a common-noun/type mention) | `resolved_node_id ∉ gold_nodes` | Graph eng |
| 6 | **Reasoning error** | Correct evidence present, answer wrong (misjoined, misread, miscounted, wrong inference) | Evidence present ∧ answer incorrect | Prompt / model |
| 7 | **Faithfulness / citation error** | Answer right or wrong, but claims unsupported by the cited chunk | `citation_support < 1` | Prompt / UX |
| 8 | **Process error** | Wrong tool, bad arguments, loop, premature stop, budget exhaustion | Trajectory scorer | Agent eng |
| 9 | **Abstention error** | Answered when it should have abstained, or vice versa | `must_abstain` mismatch | Prompt / product |

Two more that are always separate because the fix and the escalation path differ:

| # | Bucket | Detector |
|---|---|---|
| 10 | **Permissions / ACL** | Retrieved chunk outside the user's allowed set. Zero tolerance; page immediately. |
| 11 | **Safety / injection** | Canary injected content influenced the output. Zero tolerance. |

Canonical IDs for code and YAML: `corpus_gap`, `retrieval_miss`, `graph_construction_gap`, `graph_reach_miss`, `entity_link_miss`, `reasoning_error`, `citation_error`, `process_error`, `abstention_error`, plus `acl_violation` and `safety_injection`. Use these exact strings in the classifier, the annotation schema, and the histogram.

---

## 12.3 The attribution procedure

Use a documented triage priority and retain all contributing labels. These checks produce candidate diagnoses, not proof of causality. Corpus search, support, and correctness need gold labels or calibrated review; a failed search alone does not prove a corpus gap. Only classify failed runs: correct abstention on a genuinely unanswerable task is a success.

Check ACL and injection violations first. For ordinary failures, check expected behaviour and named-entity links before downstream misses. In a hybrid run with both text and graph defects, the example below prioritises a required text-retrieval miss as primary; keep graph defects as secondary. A graph-only run must not be assigned a text-retrieval miss merely because it has no text spans.

```
0. ACL or injection violation?           → ACL VIOLATION / SAFETY INJECTION
1. Task succeeded?                      → no failure bucket
2. Required abstention was violated?    → ABSTENTION ERROR
3. Required current evidence missing
   from the corpus?                    → CORPUS GAP
4. Required named-entity link wrong?    → ENTITY LINK MISS
5. Required text retrieval incomplete?  → RETRIEVAL MISS
6. Required graph path absent in graph? → GRAPH CONSTRUCTION GAP
7. Path exists but was not retrieved?   → GRAPH REACH MISS
8. Answerable but inappropriately
   abstained?                          → ABSTENTION ERROR
9. Sufficient evidence delivered but
   answer wrong/incomplete?            → REASONING ERROR
10. Claims unsupported/miscited?        → FAITHFULNESS / CITATION ERROR
11. Remaining tool/termination defect?  → PROCESS ERROR
12. None applies?                      → UNCLASSIFIED; inspect manually
```

Implement the policy over adapter-provided checks, with missing/uncertain labels routed to review rather than converted to a negative verdict:

```python
def classify(trace, case):
    if case.acl_violation(trace):
        return "acl_violation"
    if case.injection_succeeded(trace):
        return "safety_injection"
    if case.success(trace):
        return None
    if case.must_abstain:
        return "abstention_error" if case.abstention_wrong(trace) else "unclassified"
    if not case.evidence_in_corpus():
        return "corpus_gap"
    if case.has_entity_links and not case.links_ok(trace):
        return "entity_link_miss"
    if case.requires_text_evidence and case.evidence_recall(trace) < 1.0:
        return "retrieval_miss"
    if case.requires_graph_evidence:
        if not case.path_in_graph():
            return "graph_construction_gap"
        if case.node_recall(trace) < 1.0 or case.edge_recall(trace) < 1.0:
            return "graph_reach_miss"
    if case.abstention_wrong(trace):
        return "abstention_error"
    if case.evidence_present(trace) and not case.answer_correct(trace):
        return "reasoning_error"
    support = case.citation_support(trace)
    if support is not None and support < 1.0:
        return "citation_error"
    if not case.trajectory_ok(trace):
        return "process_error"
    return "unclassified"
```

**Watch the `unclassified` rate.** Above 10% and your taxonomy has a hole; open 10 unclassified traces and find the new bucket. It is usually something product-specific (language, format, tenant config) that no generic taxonomy covers.

---

## 12.4 The annotation schema

Human reviewers get the same buckets plus free text. Keep it to two clicks and one sentence.

```yaml
case_id: eval_0042
trace_id: tr_...
primary_bucket: graph_reach_miss
secondary_buckets: [entity_link_miss]
severity: high          # low | med | high | critical
user_impact: "wrong supplier named; would have triggered a refund"
expected: "Northwind and Calder"
actual: "Northwind only"
fix_hypothesis: "max_hops=1 for entity queries; raise to 2 or use path search"
```

**Severity** is what turns a histogram into a priority:

| Severity | Definition | SLA |
|---|---|---|
| Critical | Wrong answer with financial, legal, or safety consequence; or ACL leak | Same day |
| High | Confidently wrong to a user, no error surfaced | This sprint |
| Medium | Incomplete, hedged, or miscited | Backlog, ranked |
| Low | Formatting, tone, verbosity | Batch |

Fix priority = `frequency × severity weight`. Use critical security/safety incidents as overrides; do not let low observed frequency deprioritise them. Estimate frequency from representative traffic, not an error-oversampled trace queue.

---

## 12.5 The weekly failure review

Thirty minutes, same time each week, one owner. Agenda:

1. **Numbers** (5 min): success, cost, pass^3, deltas by slice.
2. **New failures** (10 min): the top 5 unclassified or newly-appeared traces, reviewed live.
3. **Bucket movement** (5 min): which bucket grew, which shrank, why.
4. **Decisions** (10 min): one fix committed, one experiment proposed, one case added.

Rules that keep it useful:

- Review **traces, not screenshots of answers.** The whole point is the process.
- Every incident produces an eval case **before** the fix.
- No bucket is allowed to be "model quality" forever. Decompose it until it is actionable.
- If a bucket has been top for three weeks and no one is assigned, escalate the staffing, not the metric.

---

## 12.6 Common misclassifications

These cost teams weeks. Watch for them.

| Symptom | Usual wrong label | Actual bucket | Tell |
|---|---|---|---|
| "It hallucinated a supplier" | Reasoning error | Graph reach miss or retrieval miss | The correct supplier is in a document that was never retrieved |
| "The graph is bad" | Graph construction | Entity link miss | Path exists, but the query hit the wrong node |
| "The model ignores context" | Reasoning error | Retrieval miss | The needed chunk was never in the context window |
| "Faithfulness dropped" | Model regression | Corpus/context change | Claim support computed against new, worse context |
| "The agent is dumb" | Reasoning error | Process error | It never called the graph tool at all |
| "Abstention is broken" | Prompt issue | Possible relevance-filter or abstention-policy failure | Compare score calibration, context relevance, and the abstention pair |

The pattern: **most "model" failures are data or retrieval failures wearing a model costume.** Chapter 6's 2×2 is the mechanical way to check.

---

## Do this on your system

- [ ] Implement the attribution procedure as code and run it over your last 100 failed cases.
- [ ] Plot the nine-bucket histogram. This is your actual roadmap.
- [ ] Check the `unclassified` rate; if > 10%, open ten traces and add a bucket.
- [ ] Add severity to every bucket and compute `frequency × severity`.
- [ ] Assign an owner per bucket. Buckets without owners do not move.
- [ ] Start the weekly 30-minute review with the four-part agenda.
- [ ] For every future incident, require a case before the fix.

## Common mistakes

- **No primary bucket.** Multi-label everything and the histogram becomes unreadable.
- **A "model quality" bucket.** Too coarse to act on; decompose it.
- **Classifying from the answer alone.** You need the trace, or you will systematically mislabel retrieval failures as hallucinations.
- **Ignoring unclassified.** It is the signal that your taxonomy is stale.
- **Prioritising by frequency alone.** A rare ACL leak outranks a common formatting nit every time.
- **No owner per bucket.** The histogram becomes a monument.

## Key takeaways

1. Nine primary buckets plus permissions and safety; exactly one primary label per failure.
2. Automate the triage policy where labels support it; confirm causes with traces and controlled replays.
3. `unclassified` is a useful metric, not a failure of the process.
4. Priority = frequency × severity, with owners per bucket.
5. Most "model" failures are data, retrieval, or linking failures.

---

**Next:** [Chapter 13 — LLM-as-judge operations](../04-operations/13-llm-as-judge-operations.md). If you are going to automate judgement, automate it properly.
