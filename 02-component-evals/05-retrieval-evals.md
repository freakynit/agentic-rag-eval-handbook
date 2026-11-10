# Chapter 5 — Retrieval evals

> **Chapter 5** · Part II: Component evals
> **Time:** 25 minutes
> **You will leave with:** the six retrieval metrics that matter, a per-sub-query scoring procedure, and the negative-rejection test almost nobody runs.

---

## 5.1 The unit of evaluation is the sub-query

A user question is not a retrieval unit. The agent's internal sub-queries are.

```
user: "Which suppliers appeared in the 2023 audit, and do we still use them?"
  ├─ sq1: "suppliers list"                    → retrieves supplier table
  ├─ sq2: "2023 audit findings"               → retrieves audit PDF
  └─ sq3: "current supplier contract status"  → retrieves contract DB
```

Score each sub-query against the evidence that *that* sub-query was supposed to surface. Then aggregate. Two numbers matter:

- **Per-sub-query recall** — the diagnostic number.
- **Question-level evidence coverage** — the product number (did we get *everything* the question needed?).

Question-level mean recall of 2/3 can hide a missing critical hop. Report each hop and the question-level all-evidence indicator, rather than treating partial recall as a task pass.

---

## 5.2 The six metrics

### 1. Evidence recall@k — the primary metric

> Of the gold evidence chunks for this sub-query, what fraction appears in the top-k retrieved?

```
evidence_recall@k = |retrieved_top_k ∩ gold_evidence| / |gold_evidence|
```

Report at a **fixed k equal to your production retrieval budget**. If production fetches 8 chunks, recall@8 is the honest number; recall@50 is fiction.

For multi-evidence sub-queries, add **all-evidence recall**, which is 1 only if *every* gold chunk is present. This is the number that predicts multi-hop success, and it is always lower than you expect.

### 2. Context precision

> Of the retrieved chunks, what fraction are actually on-evidence?

```
context_precision@k = |retrieved_top_k ∩ gold_evidence| / k
```

Here `context_precision@k` means ordinary precision at a fixed k, with unused slots counted as non-relevant. Also report precision over returned items when fewer than k are returned. This differs from Ragas' rank-sensitive [Context Precision](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/context_precision/). Exhaustive gold labels are needed for precision: unlabelled alternative evidence must be reviewed rather than automatically called irrelevant. Recall can use labelled acceptable evidence sets or fact coverage when several chunks are interchangeable.

Low precision is not fatal on its own — models handle a bit of noise — but it costs tokens, and it correlates strongly with distraction errors on long contexts. Track it as a cost/quality dial, not a gate.

### 3. Rank-sensitive quality: nDCG@k or MRR

Recall treats rank 1 and rank 8 the same. They are not. Use **nDCG@k** when you have graded relevance (gold / supporting / irrelevant) and **MRR@k** when you care about the first relevant hit. A single query has reciprocal rank; MRR is its mean over queries. See Appendix A for formulas.

### 4. Redundancy / novelty rate

> Fraction of newly retrieved chunks that are near-duplicates of already-retrieved ones.

Enterprise corpora are full of near-identical documents (v1/v2 of a policy, twelve regional copies). High redundancy means your k budget is being spent on one fact. Compute with cosine similarity ≥0.95 on chunk embeddings, or shingle overlap.

A redundancy rate above ~0.25 is a smell: either dedupe at index time or diversify at query time.

### 5. Sub-query coverage (decomposition quality)

> Did the agent generate sub-queries that together cover every evidence need in the gold set?

This is scored against gold evidence, independent of what retrieval returned:

```
coverage = |evidence_needs_addressed_by_any_subquery| / |evidence_needs|
```

It separates "the retriever is bad" from "the planner never asked the right question." These have completely different fixes.

### 6. Retrieval efficiency

> How many retrieval calls did the agent make, versus the number the reference path required?

```
efficiency = reference_retrieval_calls / actual_retrieval_calls   (capped at 1)
```

Compute efficiency only for successful runs; an agent that makes no useful calls must not earn a perfect score. If actual calls are zero, report `None` and handle no-retrieval tasks separately.

Also track **unique-evidence-per-call**: if calls 3 through 9 add no new evidence, the agent is thrashing. Chapter 9 covers the loop-detection side of this.

---

## 5.3 The metric almost nobody runs: negative rejection

Give the system a question whose answer is **not in the corpus**, where retrieval will still return k chunks (it always does). Correct behaviour is to report insufficiency, not to answer from the nearest irrelevant passage.

Score three outcomes:

| Outcome | Label | Notes |
|---|---|---|
| Retrieval returns low-relevance results, agent abstains | **correct rejection** | The target behaviour |
| Agent answers from irrelevant chunks | **false answer** | The expensive failure |
| Agent abstains on an answerable question | **over-rejection** | The annoying failure |

Report `false_answer_rate` on the unanswerable slice and `over_rejection_rate` on the answerable slice together. Tuning the decision threshold usually trades one against the other; a single number lets you lie to yourself.

A calibrated **relevance floor** can help, but it is not required for abstention: a model can reject irrelevant context even when the retriever always returns k chunks. Retrieval scores are model-specific and generally are not probabilities. Test score thresholds or a relevance classifier on answerable and unanswerable cases, then measure the downstream abstention pair.

---

## 5.4 Score at three levels

Retrieval quality is not one number because it is not one system.

| Level | Setup | What it tells you | Cost |
|---|---|---|---|
| **Retriever-level** | Fixed gold sub-queries → retriever | Index/embedding/reranker quality, isolated | Cheap, fast, great for CI |
| **Pipeline-level** | Agent-generated sub-queries → retriever | Planner + retriever interaction | Medium |
| **Agent-level** | Full run, score each actual retrieval span | Reality, including thrash and rewrites | Expensive |

Run retriever-level on every PR. Run pipeline-level nightly. Run agent-level before releases.

If retriever-level recall is good but pipeline-level is bad, your problem is the **planner** (wrong sub-queries, bad rewrites). If both are bad, it is the **index** (chunking, embeddings, metadata filters). That two-way split resolves most retrieval investigations in ten minutes.

---

## 5.5 Query rewrite fidelity

Agents rewrite queries. Rewrites can silently destroy intent.

Score the rewrite on three axes, using the original sub-query and the rewrite:

- **Constraint retention** — do entity names, dates, and filters survive? (deterministic: substring/entity match)
- **Intent preservation** — is the rewritten query still asking the same thing? (judge or reference)
- **Specificity delta** — did it get more specific (good) or vaguer (bad)? Flag rewrites that drop all named entities.

A cheap deterministic check catches most damage: if the original sub-query contains a proper noun or a date and the rewrite contains neither, flag it. Log the flag rate; it should be near zero.

---

## 5.6 A scoring implementation

```python
def score_retrieval(trace, case):
    rows = []
    for span in trace.spans_of_type("retrieval"):
        # The adapter maps each span to labelled evidence needs; it must not
        # depend on exact query-string equality for agent-generated rewrites.
        gold = set(case.gold_for_subquery(span))
        k = span.top_k
        if k <= 0:
            raise ValueError("top_k must be positive")
        got = [r.chunk_id for r in sorted(span.results, key=lambda r: r.rank)[:k]]
        got_set = set(got)

        rows.append({
            "query": span.query,
            "recall": len(set(got) & set(gold)) / len(gold) if gold else None,
            "all_evidence": gold.issubset(got_set) if gold else None,
            "precision": len(got_set & gold) / k,
            "mrr": next((1 / (i + 1) for i, c in enumerate(got) if c in gold), 0.0) if gold else None,
            "k": k,
        })

    # question-level rollup
    gold_all = set(case.all_gold_chunks)
    got_all = {r.chunk_id for s in trace.spans_of_type("retrieval")
               for r in sorted(s.results, key=lambda r: r.rank)[:s.top_k]}
    return {
        "subquery_rows": rows,
        "evidence_coverage": len(gold_all & got_all) / len(gold_all) if gold_all else None,
        "retrieval_calls": len(rows),
    }
```

Two implementation notes:

- Match on `chunk_id` when your gold labels are chunk-level. Fall back to `doc_id` only for document-level corpora, and say so in the report.
- Empty gold means recall, all-evidence, and MRR are `None`. Precision is zero only for confirmed unanswerable cases with an empty relevance set; missing labels must be recorded separately. De-duplicate results before scoring and keep retrieval-stage evidence coverage separate from evidence actually delivered to synthesis after filtering/truncation. The sketch covers text retrieval; graph evidence needs its own scorer.

---

## 5.7 Targets

Starting points for a mid-size enterprise corpus. Calibrate to your baseline; these are sanity checks, not promises.

| Metric | Unacceptable | Acceptable | Good |
|---|---|---|---|
| Evidence recall@8 | < 0.65 | 0.75–0.85 | > 0.90 |
| All-evidence recall (multi-hop) | < 0.40 | 0.55–0.70 | > 0.80 |
| Context precision@8 | < 0.25 | 0.35–0.55 | > 0.65 |
| Redundancy rate | > 0.40 | 0.15–0.25 | < 0.10 |
| Sub-query coverage | < 0.60 | 0.70–0.85 | > 0.90 |
| False-answer rate (unanswerable) | > 0.25 | 0.08–0.15 | < 0.05 |

Note the shape: high recall is achievable and cheap (bigger k, hybrid search). All-evidence recall and negative rejection are where systems actually fall down, and where the roadmap should focus.

---

## Do this on your system

- [ ] Add `chunk_id` + `rank` to retrieval spans if you have not (Chapter 4).
- [ ] Label 40 sub-queries worth of gold evidence. Compute evidence recall@k at production k.
- [ ] Compute all-evidence recall on your multi-hop slice. Expect a shock.
- [ ] Measure redundancy rate. If > 0.25, check for duplicate documents in the index.
- [ ] Run the unanswerable slice and compute false-answer rate.
- [ ] Run retriever-level vs pipeline-level on the same 40 queries and see which is worse.
- [ ] Add the rewrite-flag check (entity/date dropped) as a logged warning.

## Common mistakes

- **Reporting recall at k=50 while production uses k=8.** This is the most common dishonest number in RAG.
- **Scoring at question level only.** You lose the ability to attribute to a hop.
- **Ignoring all-evidence recall.** Averages hide that you never get the *second* document.
- **Treating precision as a gate.** It is a cost dial. Low precision wastes tokens; it rarely causes wrong answers on its own.
- **No negative test.** If every question has an answer, your system has never been tested on the thing that erodes trust fastest.
- **Chasing recall when the planner is the problem.** Check sub-query coverage before optimising the index.

## Key takeaways

1. Score per sub-query, then roll up. Question-level recall hides the broken hop.
2. Fix k at production budget and never report a different k.
3. All-evidence recall predicts multi-hop success; evidence recall does not.
4. Negative rejection is a first-class metric; evaluate a relevance floor as one possible aid to abstention.
5. Separate retriever-level from pipeline-level to distinguish index problems from planner problems.

---

**Next:** [Chapter 6 — Generation evals](06-generation-evals.md). Retrieval gave you evidence; did the model use it honestly?
