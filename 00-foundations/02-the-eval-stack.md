# Chapter 2 — The eval stack: what to measure, where, and why

> **Chapter 2** · Part I: Foundations
> **Time:** 25 minutes
> **You will leave with:** a five-layer metric map, a north-star definition, and a decision rule for when to use deterministic checks versus judged metrics.

---

## 2.1 Five layers, one direction of causality

Every measurement in this handbook belongs to one of five layers. Data flows upward; failures are diagnosed downward.

```
┌──────────────────────────────────────────────────────────────┐
│ 5. OUTCOME       Did the user's task get done?               │
│                  success rate, pass^k, cost-adjusted success │
├──────────────────────────────────────────────────────────────┤
│ 4. PROCESS       Was the path sound?                         │
│                  tool choice, args, steps, termination       │
├──────────────────────────────────────────────────────────────┤
│ 3. SYNTHESIS     Is the answer supported and complete?       │
│                  claim support, correctness, abstention      │
├──────────────────────────────────────────────────────────────┤
│ 2. RETRIEVAL     Did we get the evidence?                    │
│                  evidence recall, precision, path recall     │
├──────────────────────────────────────────────────────────────┤
│ 1. DATA/GRAPH    Is the substrate correct?                   │
│                  extraction F1, resolution, coverage         │
└──────────────────────────────────────────────────────────────┘
```

The single most common eval design error is measuring layer 5 and layer 3 only. That gives you a scoreboard with no diagnosis.

The second most common is measuring every layer with an LLM judge. Layer 1 and large parts of layer 4 are deterministic — you will get cheaper, sharper, and more stable numbers with code.

---

## 2.2 The metric map

Use this as the table of contents for the rest of the handbook. Each row is expanded in the chapter noted.

| Layer | Primary metric | Supporting metrics | Chapter |
|---|---|---|---|
| **Data / graph** | Entity extraction F1 (sampled) | entity-resolution pairwise P/R, triple correctness, corpus coverage, orphan rate, summary faithfulness | 7 |
| **Retrieval** | Evidence recall@k (per sub-query) | context precision, redundancy rate, sub-query coverage, retrieval efficiency, negative rejection | 5 |
| **Graph retrieval** | Path recall (node + edge) | entity-link accuracy, subgraph precision, hop budget, conditional correctness | 8 |
| **Synthesis** | Claim support rate (faithfulness) | correctness, completeness, citation precision, false-answer / over-rejection pair | 6 |
| **Process** | Trajectory success (tool + path + termination) | argument accuracy, step efficiency, loop rate, replan quality, pass^3 | 9 |
| **Outcome** | Task success rate | pass^k, cost/query, p95 latency, escalation rate | 10 |

Graph retrieval is a specialization of layer 2 rather than a sixth layer. It gets its own row because its metrics, data requirements, and failure modes are distinct enough to own a chapter.

---

## 2.3 Choosing a north star

"You cannot manage what you cannot measure" is half true. You cannot manage what you measure in *only one dimension* either.

Pick a **north star** and always print three guardrails beside it. Recommended pairing:

```
North star:   task success rate, per slice
Guardrails:   cost per successful task  ·  p95 latency  ·  pass^3
```

Notes on this choice:

- **Task success, not answer quality.** Answer quality is a component metric. If the agent's job is "resolve the customer's billing question," success means the question is resolved, not that the prose was faithful.
- **Cost per *successful* task**, not per task. It charges failed attempts against the successes they did not become, so an agent that fails cheaply or spins is not rewarded for a low per-query cost.
- **pass^3 as a guardrail** surfaces fragility that a mean success rate hides.

Avoid composite scores ("RAG Score = 0.4·faithfulness + 0.3·recall + …"). They are impossible to attribute, trivially gameable, and they hide slice regressions. If an executive wants one number, give them success rate and put the rest underneath it.

---

## 2.4 Three measurement instruments (and when to use each)

| Instrument | Use for | Cost | Stability | Example |
|---|---|---|---|---|
| **Deterministic check** | Anything with a checkable ground truth | ~free | Perfect | Tool called = `search_docs`; citation ID exists; JSON parses; entity ID ∈ gold set; latency < 2s |
| **Reference comparison** | Answers/evidence with labelled gold | Low | High (for exact tasks) | Exact/alias match on entity; set-F1 on supporting doc IDs; nDCG on ranked chunks |
| **Judged (LLM or human)** | Semantic quality: faithfulness, correctness on free text, trajectory soundness | Medium–high | Requires calibration | "Is every claim in this answer supported by the provided context?" |

**Decision rule.** Work down this list and stop at the first that applies:

1. Can a regex, set comparison, or ID lookup decide it? → deterministic.
2. Do you have gold labels for this exact field? → reference comparison.
3. Is the criterion semantic and you have (or can build) 100+ human labels to calibrate against? → LLM judge.
4. None of the above → you do not yet know what "good" means. Do not build a metric. Go label 30 examples first.

That last branch is doing a lot of work. Many "we need an LLM judge for X" requests dissolve once someone labels 30 cases and discovers the criterion is actually crisp.

---

## 2.5 Attribution by conditioning

The core analytical trick of this handbook: **measure quality conditional on an upstream property.** It narrows the investigation from "the answer was wrong" to hypotheses you can test with controlled replays.

Examples:

| Question | Conditioning | What it helps investigate |
|---|---|---|
| Answer wrong — retrieval or reasoning? | correctness \| path present in retrieved context | reasoning capability |
| Answer wrong — graph or LLM? | correctness \| gold path present in subgraph | LLM's ability to reason over a present path |
| Multi-hop failure — which hop? | hop-level recall for each hop | the missing hop |
| Agent wasted 12 steps — planning or execution? | step efficiency vs reference path | planning quality |
| Faithfulness dropped — model or context? | claim support rate at fixed gold context | model regression, isolated from retrieval |

Report these as **conditional rates**, always with the denominator:

```
Answer correctness overall:                    0.71  (n=200)
Answer correctness | gold evidence retrieved:  0.92  (133/145)
Answer correctness | evidence missed:           0.16  (9/55)
→ 46 of 58 failures (79%) occurred with evidence missing. Investigate retrieval first.
```

Make this a standard view, but treat it as diagnostic association. Harder cases can both retrieve worse and reason worse. Replay with gold evidence in a fixed context to test the retrieval hypothesis; also check alternative evidence and gold-label completeness.

---

## 2.6 The minimum viable eval stack

If you build nothing else, build these eight measurements. They cover roughly 80% of the diagnostic value.

1. **Evidence recall@k**, per sub-query, per slice.
2. **Context precision** — fraction of retrieved chunks that are on-evidence.
3. **Claim support rate** — fraction of answer claims entailed by delivered context; report citation support separately.
4. **Answer correctness** vs gold, per slice, with abstention split out.
5. **Trajectory success** — right tools, right args, terminated correctly.
6. **Step efficiency** — reference steps / actual steps, capped at 1 and reported only for successful runs.
7. **pass^3** on a 30-case core set.
8. **Cost per successful task and p95 latency over all attempts.**

Add graph metrics (Chapters 7–8) only if a graph is in the critical path. Add online metrics (Chapter 10) once you have traffic.

---

## 2.7 Reporting shape

Every eval report, regardless of tooling, should have the same three blocks. Teams that skip block 3 keep re-litigating the same regressions.

**Block 1 — headline (2 lines)**
```
Task success 0.74 (n=200, SE ±3.1; 95% CI 0.68–0.80)   Cost/success $0.041   p95 4.2s
vs. baseline 0.71 → +3.0 pts; paired 95% CI for Δ: −1.0 to +7.0 pts (inconclusive)
```

**Block 2 — slices (table, largest movement first)**
```
slice                 n    success   Δ vs baseline   verdict
temporal              22   0.55      -12.0           investigate (small n)
multi-hop             41   0.80      +7.0            directional gain
unanswerable          18   0.94      +2.0            flat
...
```

**Block 3 — top failure buckets (stacked counts; not necessarily exhaustive)**
```
retrieval_miss        14   ▲ 6   (new: 5 of these on the "2024 filings" corpus)
entity_link_miss       9   ▲ 3
reasoning_error        7   = 0
process_error          4   ▼ 2   (termination failures)
```

No prose. If a number moved, a human writes one line explaining why in the PR.

---

## Do this on your system

- [ ] Draw the five-layer diagram for *your* system. Which layers have zero measurements today?
- [ ] Write your north star as one sentence plus three guardrails. Circulate it.
- [ ] Build the conditional-correctness report from §2.5 for your current eval set. It takes an afternoon and usually reorders the roadmap.
- [ ] Classify every existing metric as deterministic / reference / judged. Flag any judged metric with no human calibration behind it (Chapter 13).
- [ ] Delete or demote any composite score that is being used in a release decision.

## Common mistakes

- **Judging things code can check.** "Did it call the right tool?" is not a judge question. Deterministic checks are cheaper, instant, and never drift.
- **Optimising the north star without guardrails.** Success rate alone invites agents that burn tokens and time to squeeze out a point.
- **Reporting aggregate-only.** Every metric in this handbook is per-slice or it is decorative.
- **Conditioning without denominators.** "89% when retrieved" means nothing without "of 160 cases."
- **Building metrics for criteria you haven't defined.** Label first, metric second.

## Key takeaways

1. Five layers: data/graph, retrieval, synthesis, process, outcome. Diagnose downward, report upward.
2. Pick task success as the north star; always print cost, latency, and pass^k beside it.
3. Choose deterministic → reference → judged, in that order.
4. Attribution comes from conditioning, not from staring at averages.
5. Eight measurements constitute a viable stack. Everything else is refinement.

---

**Next:** [Chapter 3 — Eval dataset design](../01-datasets-and-traces/03-eval-dataset-design.md). Metrics are cheap; labelled cases are the scarce resource.
