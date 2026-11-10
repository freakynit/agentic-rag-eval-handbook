# Chapter 3 — Eval dataset design

> **Chapter 3** · Part I: Foundations
> **Time:** 30 minutes
> **You will leave with:** a question taxonomy, a labelling protocol for gold evidence, and a sizing rule you can defend to your team.

---

## 3.1 The eval set is the asset

Metric libraries are commodities. Your labelled cases are not. A 200-case set built from real user questions, with gold evidence and expected tool paths, will outperform a 20,000-case synthetic benchmark on every decision that matters — because it measures *your* corpus, *your* users, and *your* failure modes.

Two consequences follow:

- Budget weeks, not hours, for the first set. Two engineers for one week beats one engineer for a day, every time.
- Treat it as production code: versioned, reviewed, owned, and protected by a freeze policy.

---

## 3.2 Question taxonomy

Stratify deliberately. Each row below is a slice you will report on separately, because each fails for different reasons and is fixed by different work.

| # | Slice | Example | What it stress-tests |
|---|---|---|---|
| 1 | **Single-hop lookup** | "What is the refund window for EU orders?" | Basic retrieval + faithful synthesis. Your floor. |
| 2 | **Multi-hop** | "Which of our suppliers also appeared in the 2023 audit findings?" | Decomposition, path traversal, evidence joining |
| 3 | **Aggregation / global** | "Summarise the main themes across all Q3 support tickets." | Community/global search, map-reduce faithfulness, coverage |
| 4 | **Temporal** | "What changed in the policy between v2 and v3?" | Date filtering, version awareness, staleness handling |
| 5 | **Entity disambiguation** | "Which John Smith signed the 2019 contract?" | Entity linking, resolution quality, graph precision |
| 6 | **Comparative** | "How does the Enterprise SLA differ from the Team SLA?" | Multi-document reasoning, symmetric retrieval |
| 7 | **Multi-turn follow-up** | T1: "Show me Q2 revenue." T2: "And by region?" | State retention, context carry, no re-fetching |
| 8 | **Unanswerable / out-of-corpus** | "What is our policy on lunar offices?" | Abstention calibration, negative rejection |
| 9 | **Ambiguous** | "What's the status?" | Clarification behaviour vs guessing |
| 10 | **Adversarial / poisoned** | A retrieved chunk containing "ignore previous instructions…" | Injection resistance, source trust |
| 11 | **Stale document** | Answer exists only in a superseded doc | Currency checks, conflict resolution |
| 12 | **Long-tail / rare entity** | A person mentioned once, in one PDF | Recall under sparse connectivity; graph orphan handling |

**Minimum viable stratification:** at least 20 cases in each slice you claim to support. Under 20, per-slice numbers are noise; even at 20 they are directional rather than gateable (see §3.5).

Use a representative sample for the traffic-weighted headline and a separate challenge set for rare or risky slices. If you oversample hard slices, reweight the headline using measured production shares; a raw average over that set does not estimate production success. These slices can overlap: store one primary `slice` for disjoint totals and use `tags` for additional properties such as temporal or multi-hop.

---

## 3.3 What to label (and at what granularity)

The mistake that ruins most eval sets: labelling only the answer. Here is the full case record.

| Field | Required? | Granularity | Purpose |
|---|---|---|---|
| `question` | yes | string | The input |
| `gold_answer` | for answerable cases | short string + list of required facts | Correctness, completeness |
| `gold_evidence` | yes; empty when unanswerable | list of `{doc_id, chunk_id?}`; graph paths stored separately | Retrieval recall, path recall |
| `path_constraints` | for agents | required/forbidden tools, order pairs, step budget | Tool/path correctness |
| `reference_steps` | optional | decision-step count for the reference path | Step efficiency |
| `must_abstain` | yes | bool; stale cases may instead require a current-source answer | Abstention scoring |
| `slice` | yes | enum | Reporting |
| `difficulty` | optional | 1–3 | Progress tracking, not gating |
| `source` | yes | `user` / `synthetic` / `expert` | Bias auditing |
| `created_at`, `version`, `owner` | yes | metadata | Hygiene |

### Gold evidence, precisely

Label evidence at the **chunk** level, not the document level, whenever your retriever returns chunks. Document-level labels can inflate recall and hide the fact that the right document was retrieved in the wrong section.

For multi-hop questions, label evidence **per hop**:

```yaml
question: "Which of our suppliers also appeared in the 2023 audit findings?"
gold_answer: "Northwind Logistics and Calder Freight"
gold_evidence:
  - hop: 1
    doc_id: "suppliers-2024.xlsx"
    chunk_id: "suppliers-2024#c12"
    supports: "supplier list"
  - hop: 2
    doc_id: "audit-2023.pdf"
    chunk_id: "audit-2023#c44"
    supports: "findings naming Northwind and Calder"

# Store graph nodes and typed, directed edges separately in gold_path (Appendix B).
```

Hop-level labels are what make the conditional-correctness report in Chapter 2 possible. They are also the single highest-value labelling effort you can invest in.

### Acceptable, not unique

Do not label one correct trajectory. Label *the set of acceptable ones*. Agents legitimately take different paths: search-then-graph or graph-then-search, two sub-queries or three.

Store a **partial order** rather than a fixed sequence:

```yaml
path_constraints:
  must_include:
    - {tool: "search_docs", args_contains: {query: "suppliers"}}
    - {tool: "graph_neighbors", args_contains: {entity: "Northwind"}}
  must_not_include:
    - {tool: "web_search"}          # must not go outside the corpus
  order_pairs:
    - [search_docs, graph_neighbors]
  max_steps: 10
  max_retrieval_calls: 5
  must_terminate_with: final_answer

reference_steps: 8                  # decision steps only; for step efficiency
```

Require a specific tool only when it is part of the contract being tested. A task-level case should accept a valid vector-only answer as well as a graph answer; graph-required cases belong in a component or routing test. `args_contains` means exact equality on the listed argument fields; use explicit aliases or semantic checks when equivalent query wording is allowed.

---

## 3.4 Where cases come from

| Source | Share | Strengths | Watch out for |
|---|---|---|---|
| **Real user queries** (prod logs, support tickets, sales calls) | 50–70% | Real distribution, real messiness, real failure modes | Needs PII scrubbing; skewed to what already works |
| **Expert-written** (SMEs write what *should* be answerable) | 15–30% | Covers capability gaps, tests the roadmap not the past | Expensive; SMEs write unnaturally complete questions |
| **Synthetic** (generated from documents) | 10–25% | Cheap coverage, good for graph/multi-hop scaffolding | **Systematically too easy and too clean** |

### The synthetic-data trap

LLM-generated questions inherit the phrasing of the source chunk. Real users do not. Concretely:

- Synthetic questions almost never use synonyms the retriever would fail on.
- They almost never omit a crucial constraint.
- They almost never contain typos, abbreviations, or internal jargon.
- They are usually single-hop even when prompted for multi-hop.

Mitigations that work:

1. Generate from **document clusters**, not single chunks, so the answer requires joining.
2. **Paraphrase twice**, once into casual user language, once into internal jargon/abbreviations.
3. **Remove** one constraint from 20% of generated questions to create ambiguity.
4. **Invert** 10% into unanswerable questions by removing all equivalent supporting evidence from a versioned fixture corpus. Verify that the question is actually unanswerable in that fixture.
5. Generate the *gold evidence first*, then derive the question from it. Never generate the question and then guess the evidence.

Tag every case with `source` and report metrics **split by source**. If synthetic cases score 15 points higher than real ones, your synthetic generator is miscalibrated and you should not use it for gating.

---

## 3.5 Sizing

| Purpose | Cases | Notes |
|---|---|---|
| PR smoke test | 40–60 | Must run in <5 min. Covers each supported slice plus prioritised known regressions; keep the full historical set in nightly runs. |
| Nightly regression | 150–300 | The main set. Full slice coverage; 20+ per slice needs ~240+. |
| Deep analysis / release gate | 300–600 | Add the hard slices at higher density, plus multi-turn. |
| Human-calibrated judge set | 100–200 | Doubles as judge calibration data (Ch.13). |
| Online sample | 200/week | Random prod traffic, scored automatically, reviewed weekly. |

**The 150 number.** A 150-case set gives a 95% interval of roughly ±7 points on an overall rate near 0.75, so a 5-point change needs a paired design or a larger set; an unpaired comparison needs much more. A 20-case slice gives ±18–22 points near a 50–75% rate, which is directional, not gateable. That is why you gate on the whole set and on slices large enough to support a floor, or collect more distinct cases. Repeated runs estimate within-case variation; they do not turn 20 cases into 60 independent cases. It is also why a 2-point move is usually noise and should not be gated.

Twelve slices × 20 cases is 240 cases, not 150. A 150-case set can carry 20 cases on about seven slices; choose which slices you gate on and let the rest be directional.

Do not scale to 5,000 before you have 150 that are correct. A large set with sloppy labels produces confident wrong answers.

---

## 3.6 Contamination and leakage

Three distinct problems, often conflated:

1. **Benchmark contamination** — a public benchmark's questions appeared in your model's training data. A no-context run can flag memorisation, but success alone does not prove training contamination. Prefer fresh, private, or post-training cases and document any known exposure.
2. **Answer-artifact leakage** — evaluation-only material (gold answers, rubrics, generated QA pairs) enters the searchable corpus or agent prompt. Exclude those artifacts. The answer appearing in an original source is expected in extractive RAG, not leakage; use separate multi-hop and synthesis cases to test reasoning.
3. **Eval-set overfitting** — the team has tuned prompts against the same 150 cases for six months. Fix: a working set for development, a frozen release set with controlled access, and a fresh holdout from production. Repeated release decisions also leak information about the frozen set; rotate it periodically and move any exposed cases into the working set.

Practical policy:

```
frozen/       never used during development, only in release gate
working/      used daily, expect overfitting, refresh every quarter
holdout/      new prod sample each month, never looked at until review
```

Report all three. If frozen and working diverge by more than your noise band, you have overfit and should say so out loud.

---

## 3.7 Maintaining the set

An eval set is a living thing. Assign an owner and a weekly 30-minute slot.

- **Add** a case for every production incident, before the fix. The case exists in a failing state; the fix turns it green. This is the single best source of new cases.
- **Retire** cases whose documents were deleted or whose feature was removed. Dead cases silently depress your scores and erode trust.
- **Re-label** when the corpus changes materially (a policy update, a product rename).
- **Version** everything. `eval_set_v7` with a changelog line per release.
- **Audit** 10 random cases per month against the source docs. Label rot is real and invisible.

---

## Do this on your system

- [ ] Build the 12-slice taxonomy table with your own example per slice. Mark which slices you cannot currently test.
- [ ] Pick 40 real questions, stratify them, and label gold answers + chunk-level evidence. Time it. Multiply for your target set size, then decide whether to staff it.
- [ ] Add `must_abstain` to at least 15 cases (out-of-corpus + stale-doc).
- [ ] Write the frozen/working/holdout policy in one paragraph and get it agreed.
- [ ] Check your existing synthetic cases: do they score >10 points higher than real ones? If so, stop gating on them.

## Common mistakes

- **Document-level evidence labels.** They inflate recall and hide the real misses.
- **One gold trajectory.** You will spend months arguing about legitimate alternatives. Label acceptable sets and a partial order.
- **Only answer labels.** Without evidence labels you cannot do attribution, and attribution is the whole point.
- **Synthetic-only sets.** They measure your generator, not your system.
- **Unbounded growth.** 3,000 unreviewed cases is worse than 150 reviewed ones: it is slower and you trust it more than you should.

## Key takeaways

1. The labelled set, not the metric library, is the durable asset.
2. Use the 12 question types to find coverage gaps; 20 cases per slice supports exploration, while gates need justified sample sizes.
3. Label chunk-level gold evidence per hop. This unlocks retrieval, path, and conditional-correctness metrics.
4. Accept a set of trajectories, not one. Use required/forbidden steps and a max-step budget.
5. Keep frozen, working, and holdout sets separate, and check for overfitting explicitly.

---

**Next:** [Chapter 4 — Tracing and instrumentation](04-tracing-and-instrumentation.md). You cannot score a trajectory you did not record.
