# Appendix D — Worked example

> A synthetic teaching example, traced through every layer. The company, traces, costs, and before/after results are illustrative, not measurements from a production deployment.

---

## D.1 The setup

A mid-size B2B company. Corpus: ~40k documents (contracts, audit reports, supplier records, policy PDFs). Hybrid system: vector retrieval + a knowledge graph built from those documents, with an agent that can choose either path.

Eval set: 200 cases, 12 slices. Gold path for this question is labelled. The supplier/audit case uses a frozen business-state snapshot as of 2024-12-31, so "our suppliers" refers to that snapshot rather than today.

---

## D.2 Case `eval_0042`

```yaml
slice: multi_hop
question: "Which of our suppliers also appeared in the 2023 audit findings?"
gold_answer: "Northwind Logistics and Calder Freight"
required_facts: ["Northwind Logistics", "Calder Freight"]
forbidden_facts: ["Vantage Supply"]
gold_evidence:
  - {hop: 1, doc_id: suppliers-2024.xlsx, chunk_id: "suppliers-2024#c12", supports: "supplier list"}
  - {hop: 2, doc_id: audit-2023.pdf,   chunk_id: "audit-2023#c44", supports: "findings naming both"}
gold_path:
  nodes: ["supplier:northwind", "supplier:calder", "audit_finding:AF-2231"]
  edges:
    - {from: "supplier:northwind", to: "audit_finding:AF-2231", type: appears_in}
    - {from: "supplier:calder",    to: "audit_finding:AF-2231", type: appears_in}
```

---

## D.3 What actually happened

```
step 0  plan      → 2 sub-queries: "suppliers list", "2023 audit findings"
step 1  retrieve  → sq "suppliers list"          → suppliers-2024#c12  ✅ hop 1
step 2  retrieve  → sq "2023 audit findings"     → audit-2023#c07     ❌ (c44 missed)
step 3  graph     → link "suppliers" → null (expected: common noun, not an entity; should use a type filter)
step 4  graph     → link "Northwind" (from c12) → supplier:northwind  ✅
step 5  graph     → expand up to 2 hops from Northwind → 4 neighbors incl. AF-2231
                   ❌ calder missing (edge not extracted)
step 6  reason    → "I found Northwind in the audit. Calder not mentioned."
step 7  synthesize→ "Northwind Logistics appears in the 2023 audit findings."
```

Outcome: **partial answer**, one of two required facts. Terminated at step 7: premature, because it answered with insufficient evidence rather than recovering the missing hop.

---

## D.4 Layer-by-layer scores

### Retrieval (Chapter 5)

| Sub-query | Gold chunks | Retrieved | Recall@8 | Notes |
|---|---|---|---|---|
| "suppliers list" | c12 | c12, c13, c14, plus 5 irrelevant chunks | 1.0 | |
| "2023 audit findings" | c44 | c07, c08, c09, plus 5 irrelevant chunks | **0.0** | Right document, wrong chunk — dense retrieval matched the executive summary |

```
evidence_recall (mean):        0.50
all-evidence recall:           0.0
context precision@8:           0.06  (1 of 16 top-k slots on-evidence)
text retrieval calls:          2 (reference: 2); graph calls tracked separately
retrieval efficiency:          None (task failed; call counts remain diagnostic)
sub-query coverage:            1.0   (planner was fine)
```

### Graph retrieval (Chapter 8)

| Check | Result |
|---|---|
| Entity-link accuracy | 1 of 1 intermediate named mention linked ("Northwind" → supplier:northwind) → 1.0; type/topic mentions excluded |
| Node recall | 2 of 3 gold nodes → 0.67 |
| **Edge recall** | 1 of 2 gold edges → **0.50** |
| Hop sufficiency | used 2, needed 2 from Northwind to Calder via AF-2231 → 1.0; missing edge still prevents reach |
| Subgraph precision | 2 of 5 retrieved nodes on-path → 0.40 |

The `calder → AF-2231` edge is **absent from the graph**. That is a construction gap, not an expansion limit.

### Graph construction audit context (Chapter 7)

The quarterly audit for this corpus had already reported:

```
entity recall        0.71
relation recall      0.48   ← the systemic number
hallucinated triples 0.01
pairwise resolution  P 0.96 / R 0.74
provenance coverage  0.99
corpus claim coverage 0.58
```

`eval_0042` is a textbook instance of the 0.48 relation recall: the edge exists in the document and was never extracted.

### Generation (Chapter 6)

| Metric | Score |
|---|---|
| Claim support (context) | 1.0 (the single claim is supported by what it had) |
| Citation support | 1.0 (c07 explicitly names Northwind and is cited; it omits Calder) |
| Correctness | **fail** (1 of 2 required facts) |
| Completeness | 0.50 |
| Forbidden facts present | no |
| Abstention | n/a |

Note the trap: **faithfulness is perfect.** A dashboard showing only "faithfulness 100%, no hallucinations" would declare this a clean run. It is a wrong answer.

### Trajectory (Chapter 9)

```
tool selection      5/5 calls used an appropriate tool family (vector for list, graph for relations)
argument accuracy   4/5   ("suppliers" is a type, not an entity; use a type filter)
path compliance     pass  (no forbidden tools, within step budget)
step efficiency     None  (task failed; 8 actual steps vs reference 8)
loops               0
termination         premature (answered with one of two required facts; no recovery attempt)
pass^3              0 for this case: runs 1 and 3 failed; run 2 found c44 and passed
```

### Outcome (Chapter 10)

```
task success        0
flake               yes (mixed across runs) → flip set
cost                $0.031 (reference median $0.028)
p95 for the slice   5.8s
```

---

## D.5 Attribution (Chapter 12)

Applying the procedure:

```
0. ACL/injection violation?             no
1. Task succeeded?                      no
2. Expected abstention violated?        no (answerable case)
3. Evidence in corpus and current?      yes
4. Required named-entity link correct?  yes (Northwind from c12)
5. Required text retrieval incomplete?  yes → retrieval_miss (primary)
   Secondary checks:
   - required path absent from graph   → graph_construction_gap
   - "suppliers" sent as named entity  → process_error
   - citation support < 1?             no
```

**Primary bucket: `retrieval_miss`.** Secondary: `graph_construction_gap`, `process_error`.

Three separate defects contributed. This is normal, and it is why the taxonomy needs a primary label:

| Bucket | Frequency in slice (contributing labels) | Severity | Owner |
|---|---|---|---|
| retrieval_miss | 14 | high | retrieval eng |
| graph_construction_gap | 9 | high | graph eng |
| process_error | 7 | med | agent eng |

Counts overlap: one failed run can carry a primary and secondary label. The histogram in Chapter 12 uses primary labels only; this table ranks contributing causes. Priority = frequency × severity → retrieval first, but the graph gap is the reason the fallback did not save it.

---

## D.6 The conditional table (the important slide)

For the multi-hop slice (n=50):

| Condition | n | Correctness | Reading |
|---|---|---|---|
| Path in graph **and** retrieved | 24 | 0.92 (22/24) | Higher success |
| Path in graph, **not** retrieved | 15 | 0.47 (7/15) | Investigate expansion / linking |
| Path **not** in graph | 11 | 0.27 (3/11) | Investigate extraction; check text fallback |

Two conclusions, both actionable and both invisible in the aggregate 0.64:

1. Correctness is higher when the path is retrieved (22/24). Confirm the evidence-delivery hypothesis by replaying failures with gold context; the conditional table alone is not causal proof.
2. 26 of 50 cases lack a retrieved gold graph path, but 10 of those still succeed via text or another route. Of 18 failures, 16 occur in those two groups. Graph construction and reach deserve investigation without treating missing paths as automatic task failures.

---

## D.7 What the team did

In order, with the evidence that justified each:

1. **Chunk-level reranking for audit PDFs.** The retriever matched the executive summary over the findings section. Added a cross-encoder on the top 30. `evidence_recall` on the temporal+multi-hop slices: 0.50 → 0.78. Cost: +$0.002/query.
2. **Two-pass relation extraction on tables and PDFs with `appears_in`, `party_to`, `reports_on` predicates.** `relation_recall` 0.48 → 0.61 on the audit. Build cost +18%. This is the expensive one, and the gain justified it because the table identified 11/50 cases with a missing graph path, and controlled replays confirmed a benefit for affected failures.
3. **Common-noun handling.** When a mention is a common noun or a type, use a type filter or fall back to vector retrieval instead of calling the entity linker. Removed 6 of 7 wasted graph calls.
4. **Added eval_0042 to the permanent regression set.** It now runs in the 40-case smoke suite, and its `pass^3` status is on the weekly page.

Result on the multi-hop slice after two sprints: 0.64 → 0.80, at +$0.003/query and +1% p95.

---

## D.8 The second case: where the graph actually pays

Case `eval_0061`, same corpus, slice `entity_disambiguation`:

> "Which John Smith approved the 2024 framework agreement?"

Vector-only RAG retrieves three documents mentioning a John Smith (sales, legal, engineering) and answers with the sales VP, who is mentioned most often. Graph path: `person:js-8812 (legal, GC)` → `approved` → `contract:FW-2024` → correct.

| Metric (20 distinct cases × 5 runs per arm; rounded means) | Vector RAG | With graph |
|---|---|---|
| Correctness | 0.62 | 0.84 |
| Claim support | 0.88 | 0.95 |
| Cost/query | $0.005 | $0.026 |

On this slice (n=20), the graph buys **+22 points** for **5.2x cost per query**, and the win concentrates on questions where a name is shared or a relation is load-bearing.

This motivates testing routing. In this synthetic scenario, independently measured traffic puts the slice at 10%; its 20 evaluation cases alone do not establish that share. Validate router errors, TCO, and the value of extra successes before adopting it. See Chapter 11 for the full slice mix and the hybrid cost argument.

---

## D.9 What this example is meant to show

1. **Perfect faithfulness and a wrong answer coexist comfortably.** Never report one without the other.
2. **The aggregate hides everything.** 0.64 on the slice looked like a model problem; the conditional table showed that most failures coincided with missing graph paths, motivating controlled replays.
3. **The failure was three different bugs** in three different components, with three different fixes and three different costs.
4. **Failures are often multi-causal.** Pick a primary bucket anyway, or the histogram becomes unreadable.
5. **The routing decision came from a slice table**, not from an argument about architecture.
6. **The flaky case was the most informative one.** It found the right chunk once, which proved the signal existed and the pipeline was merely unreliable.
