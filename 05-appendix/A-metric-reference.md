# Appendix A — Metric reference

> Quick lookup. Retrieval metrics assume a stated budget `k`; `k` in pass^k means trials. Report per slice with denominators. Empty denominators are `None` unless a metric explicitly defines otherwise. Targets are illustrative product choices, not benchmark norms.

**Instrument key:** `D` deterministic · `R` reference comparison · `J` judged. Combinations (`R/J`, `D/R/J`) mean the metric can be computed at more than one level; use the strongest instrument your labels support.

---

## A.1 Data and graph construction

| Metric | Instr. | Definition | Formula / method | Data needed | Target |
|---|---|---|---|---|---|
| Chunk self-containment | R | Chunks interpretable without neighbours | Human 50-sample rate | 50 chunks | > 0.85 |
| Entity precision | J/R | Extracted entities that are real and well-typed | correct / extracted | 200 sampled entities | > 0.85 |
| Entity recall | R | Gold entities that were extracted | found / gold | 200 entities + gold | > 0.70 |
| Type accuracy | D | Correct entity type | correct / total | Gold types | > 0.90 |
| Relation precision | J/R | Triples correct incl. direction | correct / extracted | 200 triples | > 0.80 |
| Relation recall | R | Gold relations extracted | found / gold | Gold triples | > 0.55 |
| Hallucinated triple rate | J | Triples not supported by any chunk | unsupported / total | 200 triples | < 0.02 |
| Pairwise resolution precision | R | Merged pairs that are truly same | TP / (TP+FP) | 150 labelled pairs | > 0.95 |
| Pairwise resolution recall | R | True duplicate pairs merged | TP / (TP+FN) | 150 labelled pairs | > 0.80 |
| Pure-cluster rate | R | Clusters containing one real entity (not standard weighted purity) | pure / total | Sampled clusters | > 0.95 |
| Fragmentation rate | R | Gold entities split across nodes | split / gold | Gold entity list | < 0.15 |
| Provenance coverage | D | Nodes/edges with ≥1 source chunk | linked / total | Graph export | > 0.98 |
| Provenance correctness | J/R | Cited chunk states the relation | correct / sampled | 100 edges | > 0.95 |
| Community stability | D | Partition agreement at the same hierarchy level | Adjusted Rand/MI, or matched-community Jaccard | Two runs | > 0.80 |
| Summary support | J | Summary claims supported by members | supported / claims | 50 summaries | > 0.90 |
| Corpus claim coverage | R/J | Corpus claims represented in graph | covered / gold claims | 50 chunks | > 0.60 |

## A.2 Retrieval

| Metric | Instr. | Definition | Formula | Data needed | Target |
|---|---|---|---|---|---|
| Evidence recall@k | R | Gold chunks retrieved | `len(top_k ∩ gold) / len(gold)` | chunk-level gold | > 0.85 |
| All-evidence recall | R | All gold chunks retrieved | `gold ⊆ top_k` | chunk-level gold | > 0.70 |
| Context precision@k | R | Retrieved chunks on-evidence | `len(top_k ∩ gold) / k` | chunk-level gold | > 0.40 |
| nDCG@k | R | Rank-weighted relevance | DCG@k / ideal_DCG@k; None if ideal=0 | graded labels | > 0.60 |
| MRR@k | R | Mean reciprocal rank of first relevant hit | mean(1/rank); 0 if no hit within k | binary labels | > 0.65 |
| Redundancy rate | D | Near-duplicate retrieved chunks | near-duplicate returned items / returned items; count each item at most once | embeddings | < 0.25 |
| Sub-query coverage | R | Evidence needs addressed by sub-queries | covered needs / needs | gold evidence | > 0.85 |
| Retrieval efficiency | R | Calls vs reference, successful runs only | min(1, ref / actual); None if actual=0 | reference path | > 0.80 |
| Rewrite constraint retention | D | Entities/dates preserved in rewrite | preserved / present | rewrite span | > 0.95 |
| False-answer rate | J/D | Answered when unanswerable | false answers / unanswerable | `must_abstain` | < 0.15 |
| Over-rejection rate | D | Abstained when answerable | abstains / answerable | answerable set | < 0.10 |

For graded labels `rel_i`, `DCG@k = Σ_{i=1..k} (2^rel_i − 1) / log2(i+1)`. Compute ideal DCG from the same judged candidate universe sorted by relevance, and state the gain convention. Missing relevance labels require review. Context precision here is ordinary fixed-budget precision@k, not Ragas' rank-sensitive Context Precision.

## A.3 Graph retrieval

| Metric | Instr. | Definition | Formula | Data needed | Target |
|---|---|---|---|---|---|
| Top-1 entity-link accuracy | R | Mention → gold node first | correct / mentions | 150 mentions + gold | > 0.90 |
| Wrong-node rate | R | Linked to a real but wrong node | wrong / mentions | same | < 0.05 |
| Null-link rate | D | No link produced | null / mentions | same | < 0.08 |
| Multi-mention completeness | R | Query entities linked | linked / named | comparative cases | > 0.90 |
| Node recall | R | Gold path nodes retrieved | `len(nodes ∩ gold) / len(gold)` | gold paths | > 0.85 |
| Edge recall | R | Gold path edges retrieved | `len(edges ∩ gold) / len(gold)` | gold paths | > 0.70 |
| Path recall (strict) | R | Both node and edge recall = 1 | exact subset | gold paths | > 0.60 |
| Subgraph precision | R | Nodes/edges on or supporting a valid path | report node and edge precision separately | gold paths | > 0.30 |
| Hub contamination | D | Subgraph from p99-degree nodes | hub nodes / retrieved | degree stats | < 0.25 |
| Hop sufficiency | R | hops_used ≥ hops_needed | cases / total | hop labels | > 0.90 |
| Community selection recall | R | Relevant communities retrieved | found / relevant | labelled communities | > 0.75 |
| Map faithfulness | J | Per-community notes supported | supported / claims | community text | > 0.90 |
| Reduce faithfulness | J | Synthesis supported by map notes | supported / claims | map notes | > 0.90 |
| Correctness \| path present | R+J | Correctness conditioned | correct / path-present | traces + gold | > 0.85 |

## A.4 Generation

| Metric | Instr. | Definition | Formula | Data needed | Target |
|---|---|---|---|---|---|
| Claim support (context) | J | Claims entailed by any chunk | supported / claims | answer + chunks | > 0.92 |
| Citation support | J | Claims entailed by cited chunk | supported / claims | answer + citations | > 0.85 |
| Strict support | J | All required claims fully cited, no unsupported qualifiers | passing answers / answers requiring support | answer + citations | > 0.80 |
| Correctness | D/R/J | Answer matches gold | per task type | gold answer | > 0.75 |
| Completeness | R | Required facts present | present / required | required-facts list | > 0.80 |
| Fully correct | R+J | Correct ∧ complete | both | both | > 0.65 |
| Calibration error (ECE) | D | Weighted gap between bin accuracy and probability confidence | Σ (n_bin/n) × abs(acc_bin − conf_bin) | calibrated probabilities, bin sizes | < 0.10 |
| Distractor flip rate | R | Correct answer becomes incorrect with 1 distractor | harmful flips / initially correct cases | injected distractor | < 0.15 |
| Paraphrase consistency | R | Required facts stable across paraphrases | stable / cases | 3 paraphrases | > 0.85 |

Abstention is deliberately absent as a single metric: report the false-answer and over-rejection pair from A.2, plus the calibration curve.

## A.5 Process and outcome

| Metric | Instr. | Definition | Formula | Data needed | Target |
|---|---|---|---|---|---|
| Tool selection accuracy | D | Right tool per step | correct / steps | reference tools | > 0.95 |
| Argument accuracy | D | Right args per call | correct / calls | expected args | > 0.92 |
| Path compliance | D | Constraints satisfied | pass / cases | path constraints | > 0.92 |
| Step efficiency | R | Reference steps / actual, successful runs only | capped at 1; None if undefined | reference steps | > 0.75 |
| Loop rate | D | Runs with ≥1 loop detector | loops / runs | traces | < 0.08 |
| Wasted steps | D/J | Steps confirmed unnecessary for task progress | count; no-new-evidence flags require review | traces | median < 1.5; report p95 |
| Premature termination | D/R | Answered with insufficient evidence | cases / total | evidence check | < 0.08 |
| Budget exhaustion | D | Hit max_steps | runs / total | traces | < 0.04 |
| Recovery rate | D | Recovered from injected failure | recovered / injected | fault injection | > 0.80 |
| pass^k | R | All k scheduled runs succeed | all-pass cases / total; for n≥k see Ch.9 | k runs | > 0.85 (k=3) |
| Flake rate | R | Mixed outcomes across runs | flaky / cases | k runs | < 0.05 |
| Task success | D/R/J | Product-defined conjunction | per Ch.10 | all of the above | > 0.75 |
| Cost per successful task | D | Total cost / successes | Σcost / success | cost spans | set per product |
| p95 latency | D | 95th percentile end-to-end over all attempts | percentile | timings | < SLO |

---

## A.6 Reading the targets

- They are **starting points** for a mid-size enterprise corpus with a capable model and judge. Calibrate to your baseline: a target you never hit is a decoration.
- **Recall and support rates are easy to move** (bigger k, stricter prompt). **All-evidence recall, edge recall, citation support, abstention pairs, and pass^k are hard** — they are where the roadmap should focus.
- Report **n** with every number. A rate without a denominator is an anecdote.

---

## A.7 A note on composite scores

Do not build one. If forced, use:

```
quality = task_success
guardrails = (cost_per_success, p95_latency, pass^k, acl_violations)
```

and never a weighted sum of component metrics. Weighted sums hide the component that broke, which is the only thing you needed.
