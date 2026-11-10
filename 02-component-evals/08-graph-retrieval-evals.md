# Chapter 8 — Graph retrieval and reasoning evals

> **Chapter 8** · Part II: Component evals
> **Time:** 30 minutes
> **You will leave with:** an entity-linking accuracy test, path recall, and the conditional metric that separates "graph failed" from "LLM failed to reason."

---

## 8.1 Two questions, not one

A GraphRAG query can fail because:

1. The graph did not contain the answer (construction — Chapter 7), or
2. The retrieval did not reach the part of the graph that did (this chapter).

Almost every GraphRAG evaluation collapses these. You need to keep them apart, because the fixes are in different teams and different budgets.

### The chain you are scoring

```
mention in query
   └─▶ entity linking       "Northwind" → supplier:northwind (correct?)
       └─▶ seed set         are the right nodes seeds?
           └─▶ expansion    neighbours / paths / communities
               └─▶ subgraph does it contain the gold path?
                   └─▶ synthesis  can the LLM use it?
```

Each arrow is measurable. Most teams measure only the last.

---

## 8.2 Entity linking accuracy — the bottleneck

This is the most under-measured step in GraphRAG and frequently the largest error source. A perfect graph is useless if the query resolves "Northwind" to the wrong node.

### The test

Take 150–200 **named-entity mentions** from real queries or intermediate retrieval results. Report query-origin and intermediate-origin mentions separately. Common nouns and type mentions belong in a separate type-filter test, not here. For each, record:

- The surface form as it appeared.
- The gold node ID(s).
- What your linker returned, with `link_score` and candidate count.

| Metric | Definition | Target |
|---|---|---|
| **Top-1 link accuracy** | Fraction where the top candidate is a gold node | > 0.90 |
| Top-3 recall | Fraction where a gold node is in the top 3 | > 0.97 |
| **Wrong-node rate** | Fraction resolved to a real but incorrect node | < 0.05 |
| Null rate | Fraction of named-entity mentions where linking failed entirely | < 0.08 |
| Ambiguity handling | Of genuinely ambiguous mentions, fraction where it asks or disambiguates | > 0.70 |

**Wrong-node is worse than null.** A null lets the system fall back to vector search; a wrong node sends the traversal into a confidently irrelevant subgraph, and the answer will be fluent and wrong. Track the two separately and bias your linker toward abstaining.

### Where linking fails

- **Common names** — "John Smith" with 40 candidates.
- **Internal shorthand** — "the Atlas project", "Q3 deck", account nicknames.
- **Products vs. companies** — same token, different type.
- **Mentions with no node** — the entity exists in the corpus but was never extracted (a construction failure leaking into linking).
- **Multi-entity queries** — "compare Northwind and Calder" needs two links; the linker often resolves only the first.

That last one deserves its own metric: **multi-mention completeness** — of the entities named in a query, what fraction got linked. For comparative questions this is the whole task.

### Cheap mitigations worth testing

- Type-constrained linking (if the query asks "which supplier", restrict candidates to `supplier` nodes).
- Context-window linking (use surrounding query terms, not the mention alone).
- Ask-before-traverse on low `link_score` and ≥3 close candidates.

Measure linking accuracy before and after each; these are usually bigger wins than swapping the LLM.

---

## 8.3 Path recall — the core graph metric

> Of the gold reasoning path for this question, what fraction of its nodes and edges did the retrieved subgraph contain?

```
node_recall = |retrieved_nodes ∩ gold_nodes| / |gold_nodes|
edge_recall = |retrieved_edges ∩ gold_edges| / |gold_edges|
```

Match edges by canonical `(source, relation_type, target)` including direction and any task-relevant temporal qualifiers. Score against any labelled acceptable path/evidence set; one gold path may not be unique. Empty gold sets are `None`, not division by zero.

Path recall is the pair `(node_recall, edge_recall)`, not a single blended number. If you must collapse it, state the weighting; averaging nodes and edges hides which one failed.

For multi-hop questions, label the gold path in the eval case:

```yaml
gold_path:
  nodes: ["supplier:northwind", "audit_finding:AF-2231", "policy:vendor-review-2024"]
  edges:
    - {from: "supplier:northwind", to: "audit_finding:AF-2231", type: "appears_in"}
    - {from: "audit_finding:AF-2231", to: "policy:vendor-review-2024", type: "triggers"}
```

Report both:

- **Node recall** — did the subgraph contain the right entities?
- **Edge recall** — did it contain the right *connections*? Edge recall is lower and more meaningful; a set of correct nodes with no connecting edges cannot answer a multi-hop question.

Typical result on a 2-hop question set: node recall 0.85, edge recall 0.55. That gap is the reason answers are shallow.

---

## 8.4 The conditional metric that ends the blame game

```
answer_correctness | gold_path_present_in_subgraph
answer_correctness | gold_path_absent
```

Example report:

```
Overall correctness:                    0.62  (112/180)
  | path present in subgraph:           0.88  (97/110)
  | path absent:                        0.21  (15/70)
```

Reading: 55 of 68 failures (81%) occurred when the path was absent. Prioritise construction and retrieval investigation, but do not infer that all 55 were caused by retrieval. Case difficulty and alternative text evidence can confound the comparison. Replay failures with the gold path and its provenance supplied to synthesis to test whether evidence delivery fixes them.

Now the three-way split, which is even more useful:

| Condition | Correctness | Interpretation |
|---|---|---|
| Path in graph **and** in subgraph | high | Working |
| Path in graph, **not** in subgraph | low | **Retrieval/expansion failure** — fix hops, seeds, linking |
| Path **not** in graph | low | **Construction failure** — fix extraction/coverage (Ch.7) |

That table is the single most valuable artifact in a GraphRAG evaluation. It tells you which team to page.

---

## 8.5 Expansion quality

Neighbourhood expansion is where precision dies. Pulling 3 hops of neighbours on a hub node yields hundreds of irrelevant nodes and poisons the context.

| Metric | Definition | Target |
|---|---|---|
| Subgraph precision | Fraction of retrieved nodes/edges on the gold path or directly supporting it | > 0.30 (low is normal; watch the trend) |
| Hop budget adherence | Fraction of queries within the configured max hops | ~1.0 |
| Marginal evidence gain per hop | New on-path evidence added by hop n | Inspect zero-gain tails; useful evidence may appear only at a later hop |
| Hub contamination | Fraction of subgraph coming from nodes with degree > p99 | < 0.25 |

### The hop analysis you should run once

For each multi-hop question, record the **minimum hops needed** and the **hops actually used**. Then bucket:

- `used < needed` → under-expansion. Answers are incomplete. Raise max_hops or improve path search.
- `used > needed` → over-expansion. Answers get diluted and expensive. Add pruning or a relevance threshold.
- `used == needed` → depth is sufficient; still check seeds, edges, provenance, and actual evidence coverage.

Depth alone is not a correctness check. A one-hop expansion from the wrong seed can satisfy the budget while missing every gold edge.

Over-expansion is more common than people think because agents "search a bit more to be safe," and it directly causes the cost and distractor problems in Chapters 6 and 11.

---

## 8.6 Global search evals (communities)

If your system does community/global search (summarise across a corpus, "what are the themes"), it needs its own metrics. Global and local fail for different reasons and must never be blended into one score.

| Metric | Definition |
|---|---|
| Community selection precision | Of the communities retrieved, how many actually contain relevant content |
| Community selection recall | Of the communities that should have been consulted, how many were |
| Map-phase faithfulness | Are intermediate per-community notes supported by their community? |
| Reduce-phase faithfulness | Is the final synthesis supported by the map notes? Also audit final claims against original sources to catch inherited errors. |
| Coverage | Does the answer represent all selected communities, or only the biggest? |
| Cross-community synthesis | For questions spanning communities, are the connections actually drawn? |

The map-reduce structure gives you two distinct faithfulness scores, and they fail independently. Reduce-phase drift (the model inventing a theme that no map note supports) is common and invisible if you only check the final answer against documents.

**Selection is usually the failure.** If the retriever picks 8 of 40 communities and misses the two that matter, no amount of synthesis quality saves you. Build a labelled set: for 40 global questions, label the relevant community IDs. Compute selection recall. This is a half-day of work and it is almost never done.

---

## 8.7 Robustness of the graph retrieval

- **Distractor nodes** — add one plausible wrong neighbour. Does the answer incorporate it? (Same test as Chapter 6, but at the graph layer.)
- **Edge deletion** — remove one on-path edge. Does the system degrade gracefully (retrieve adjacency and note uncertainty) or confidently answer from half a path? Graceful degradation is a real, testable property.
- **Alias stress** — ask with an alias the linker has not seen. Does it degrade to vector search, or fail hard?
- **Empty neighbourhood** — a node with no useful neighbours. Does it say so, or hallucinate links?

Each of these is 20 cases. Together they are the difference between a demo and a system.

---

## 8.8 Targets

| Metric | Unacceptable | Acceptable | Good |
|---|---|---|---|
| Top-1 entity link accuracy | < 0.80 | 0.88–0.95 | > 0.97 |
| Wrong-node rate | > 0.12 | 0.03–0.07 | < 0.02 |
| Edge recall (2-hop) | < 0.40 | 0.55–0.70 | > 0.80 |
| Correctness \| path present | < 0.75 | 0.85–0.92 | > 0.95 |
| Correctness \| path absent | — | should be *reported*, not targeted | — |
| Community selection recall | < 0.50 | 0.65–0.80 | > 0.90 |
| Hub contamination | > 0.40 | 0.15–0.25 | < 0.10 |

---

## Do this on your system

- [ ] Build the 150-mention linking test. Report top-1, wrong-node, and null rates separately.
- [ ] Add multi-mention completeness (comparative queries) as its own metric.
- [ ] Label gold paths for 40 multi-hop cases (nodes + edges).
- [ ] Compute node recall and edge recall. Expect edge recall to be much worse.
- [ ] Run the three-way conditional table (path in graph / in subgraph / neither).
- [ ] Run the hop analysis: minimum needed vs actually used, bucketed.
- [ ] Label relevant communities for 40 global questions and compute selection recall.
- [ ] Run the edge-deletion and distractor-node robustness tests.

## Common mistakes

- **Reporting one "GraphRAG accuracy" number.** It cannot distinguish linking, expansion, and synthesis failures.
- **Not measuring entity linking at all.** It is usually the biggest single error source.
- **Conflating wrong-node with no-link.** They need different fixes and different fallbacks.
- **Blending global and local search.** They fail in opposite directions and cancel out.
- **Ignoring edge recall.** Correct nodes with missing connections read as "the model can't reason" when it had no path.
- **Blaming the LLM when the path was never retrieved.** Check the conditional first.

## Key takeaways

1. Entity linking is the most under-measured and often the largest error source. Bias toward null over wrong.
2. Path recall (especially edge recall) is the core graph-retrieval metric.
3. Compare correctness with and without a retrieved path, then use gold-context replays to test graph versus synthesis hypotheses.
4. Over-expansion is as common as under-expansion and directly drives cost and distraction.
5. Global search needs community-selection recall, not just synthesis quality. Selection is usually what fails.

---

**Next:** [Chapter 9 — Trajectory evals](../03-system-evals/09-trajectory-evals.md). The agent's process, step by step.
