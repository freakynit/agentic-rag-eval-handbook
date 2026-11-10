# Chapter 7 — Graph construction evals

> **Chapter 7** · Part II: Component evals
> **Time:** 30 minutes
> **You will leave with:** a stage-by-stage construction audit, a sampled extraction protocol, and the resolution test that catches the errors no one notices.

---

## 7.1 The graph is a model output

In vector RAG, embeddings are learned representations and preprocessing can also introduce errors. In many GraphRAG systems, entities, relations, and summaries are additionally **inferred** by an LLM or extraction model. Curated knowledge graphs and structural document graphs need audits tailored to their construction process. That makes it:

- **Wrong sometimes**, silently.
- **Expensive**, once and forever (rebuilds, incremental updates).
- **Load-bearing**, in a way a vector index is not — a missing edge is a missing capability.

Yet almost every GraphRAG evaluation you will find online measures only the final answer. This chapter exists because construction quality is the most common root cause of GraphRAG disappointment, and the least measured.

### The pipeline you are auditing

```
documents
  └─ 1. chunking            chunk boundaries, overlap, metadata
      └─ 2. extraction      entities, types, relations, claims
          └─ 3. resolution  merge aliases, split collisions
              └─ 4. linking chunk ↔ node ↔ edge provenance
                  └─ 5. structure  communities / clusters / hierarchy
                      └─ 6. summaries  community or node summaries
```

Each stage has its own eval and its own failure signature. Score them separately; a blended "graph quality" number is useless.

---

## 7.2 Stage 1 — Chunking

Boring, under-measured, and upstream of everything.

| Check | Method | Failure signature |
|---|---|---|
| Chunk self-containment | Sample 50 chunks; can a human tell what entity/topic each is about without neighbours? | Entities referenced only by pronoun ("he", "the company") |
| Boundary integrity | Sample 50; do tables, lists, and code blocks get split? | Split tables → numeric extraction errors |
| Metadata preservation | Assert every chunk carries `doc_id`, `section`, `date`, `authority` | No date → temporal questions unanswerable |
| Overlap | Report overlap ratio | Too low → facts split across chunks; too high → duplicate triples |
| Size distribution | Histogram of tokens | Fat tail → extraction quality collapses on long chunks |

**Self-containment rate** target: > 0.85. Use the 50-chunk audit to identify boundary and context defects before comparing extraction models.

---

## 7.3 Stage 2 — Extraction

### The sampling protocol

For precision, sample **200 extracted entities and 200 extracted relations**, stratified across documents. For recall, independently sample source chunks and annotate all in-scope entities and relations, including ones the extractor missed. Sampling only extracted items cannot estimate recall. State the ontology, document strata, and sampling weights.

| Metric | Definition | Method |
|---|---|---|
| Entity precision | Of extracted entities, fraction that are real, well-typed, and worth having | Human or judge against source chunk |
| Entity recall | Of gold entities in the sampled chunks, fraction extracted | Human annotates gold entities for the sample |
| Type accuracy | Fraction with the correct entity type | Deterministic vs gold types |
| Relation precision | Of extracted triples, fraction correct (subject, predicate, object, direction) | Human or judge |
| Relation recall | Of gold relations, fraction extracted | Human annotation |
| Attribute accuracy | Fraction with correct attached attributes (dates, amounts, roles) | Deterministic where possible |
| **Hallucinated triple rate** | Triples not supported by any chunk | High-severity; target < 2% |

### What "good" actually looks like

Measure precision and recall separately on your corpus; extraction quality depends strongly on ontology, document format, model, and annotation rules. Do not treat an uncited range as a benchmark norm. [GraphRAG-Bench](https://arxiv.org/abs/2506.05690) provides a pipeline-level evaluation design to adapt.

Missing relations constrain graph-only reasoning when those relations are necessary. They are not a universal ceiling on task success: equivalent graph paths, original text, or vector fallback may still supply the evidence. Report success with and without those fallbacks.

### Where extraction fails, specifically

- **Pronouns and implicit subjects** — "she approved it" after a 400-token gap.
- **Negation and modality** — "did not renew", "may acquire" extracted as positive relations.
- **Temporal qualifiers** — "was CEO until 2023" becomes a timeless `is_ceo` edge.
- **N-ary relations** — a contract with buyer, seller, date, and amount collapses into lossy binary edges.
- **Tables** — row/column headers lost, producing garbage triples.
- **Long-tail entities** — rare names get dropped or genericised.

Each of these deserves a small targeted test set (20–30 chunks each) with gold triples. Negation and temporal handling in particular cause confident wrong answers that look authoritative.

---

## 7.4 Stage 3 — Entity resolution

The most damaging and least visible stage. Two failure directions:

- **Over-merging** — "John Smith (CEO)" and "John Smith (engineer)" become one node. The graph now asserts false facts, and they propagate to every neighbour.
- **Under-merging** — "IBM", "I.B.M.", "International Business Machines", and "Big Blue" become four nodes. Degree is fragmented; multi-hop recall collapses.

### The resolution test

Build a labelled set of **150–200 pairs** drawn from your actual graph, with a deliberate mix:

- ~40 true duplicates with easy aliases (acronym, punctuation)
- ~30 true duplicates with hard aliases (nickname, historical organisation name, transliteration)
- ~40 same-name-different-entity collisions
- ~40 near-miss hard negatives (same surname, adjacent product names)

A subsidiary and its parent are distinct entities; represent their relationship instead of merging them.

Then compute pairwise precision and recall at your merge threshold, and plot the threshold curve. This deliberately balanced challenge set measures stress performance, not production precision. Use a representative or properly weighted pair sample for deployment estimates and audit candidate-blocking recall separately. Typical finding: the default threshold over-merges on person names and under-merges on organisation names — opposite errors, from the same setting. Person and org resolution usually need **different** thresholds and different blocking rules.

| Metric | Definition | Target |
|---|---|---|
| Pairwise precision | Of pairs merged, fraction truly same | > 0.95 (over-merge is worse) |
| Pairwise recall | Of true duplicate pairs, fraction merged | > 0.80 |
| Pure-cluster rate | Fraction of sampled clusters containing one real entity (not standard weighted cluster purity) | > 0.95 |
| Fragmentation rate | Gold entities split across ≥2 nodes | < 0.15 |

Bias toward precision. An over-merged cluster creates false multi-hop paths and is far harder to detect downstream than a fragmented one. When in doubt, do not merge, and let retrieval's neighbourhood expansion compensate.

### Downstream symptom check

You can spot resolution problems without labels:

- **Degree outliers** — a person node with 400 neighbours in a 10k-node graph is probably a merge of several people. Print the top 20 by degree and eyeball them.
- **Cluster count vs. entity count ratio** — pathologically low means over-merging.
- **Shared-attribute contradiction** — flag incompatible attributes under your ontology and time model. Multiple headquarters or co-CEOs can be legitimate; contradictions are review signals unless the schema rules them out.

---

## 7.5 Stage 4 — Provenance and linking

Every node and edge must trace back to the chunk(s) that justify it. Without provenance:

- You cannot cite graph-derived answers.
- You cannot re-extract one document when it changes.
- You cannot audit a wrong answer.

Checks:

- **Link coverage**: fraction of nodes/edges with ≥1 provenance chunk. Target > 0.98.
- **Link correctness**: sample 100 edges; does the cited chunk actually state that relation? Target > 0.95.
- **Chunk→entity recall**: of entities in a gold chunk, how many are linked to it? This is what makes "find the document behind this node" work.

---

## 7.6 Stage 5 — Structure (communities / clusters)

If you use community detection for global search (Leiden, Louvain, or similar):

| Metric | What it tells you | Watch for |
|---|---|---|
| Modularity | How well-separated communities are | High modularity can mean tiny, useless clusters |
| Community size distribution | Balance | One giant community swallowing 40% of the graph is a failure |
| Coverage | Fraction of nodes in a non-trivial community | Orphans never appear in global answers |
| Stability | Run detection twice with a seed change; how much do memberships move? | Unstable communities → non-reproducible answers |

**Stability matters more than modularity.** If a re-run reshuffles communities, your global answers are non-deterministic for reasons unrelated to the model, and you cannot debug regressions. Compare partitions at the same hierarchy level using adjusted Rand index or adjusted mutual information. If using Jaccard, first match communities across runs, state the matching and size weighting, and report unmatched communities; raw community IDs are arbitrary. Calibrate a stability threshold against downstream changes.

---

## 7.7 Stage 6 — Summary faithfulness

Community (or node) summaries are generated text that then feeds the answer model. They are a second hallucination surface, and a bad one because the summary is reused for many queries.

Score like Chapter 6, but with the community's member nodes/chunks as the context:

```
summary_support_rate = supported_claims / total_claims
```

Target > 0.90, with **zero tolerance for invented entities or numbers**. Sample 50 summaries stratified by community size — small communities are usually fine, large ones drift.

Also measure **coverage**: what fraction of a community's salient facts appear in the summary? Summaries that are faithful but omit half the community produce confident, incomplete global answers.

---

## 7.8 Coverage against the corpus

The question no per-stage metric answers: **how much of the corpus's knowledge made it into the graph?**

```
claim_coverage = represented_claims_in_graph / gold_claims_in_sampled_chunks
```

Sample 50 chunks, enumerate their factual claims by hand (or with a careful judge), and check how many are represented. Interpret coverage against your ontology and task distribution. Missing corpus claims do not translate directly into a task-success ceiling: tasks need different claims, and fallback retrieval can bypass the graph.

This number should be on the same dashboard as answer quality, because it explains it.

---

## 7.9 The construction quality/cost curve

Construction is not free, and quality is tunable. Measure the curve rather than picking one setting:

The following are illustrative experiment hypotheses, not measured gains to expect.

| Lever | Quality effect | Cost effect |
|---|---|---|
| Stronger extraction model | +10–20 pts entity recall | 3–10x build cost |
| Smaller chunks | Better precision, worse context | More chunks, more calls |
| Two-pass extraction (extract, then verify) | +5–10 pts precision | 2x build cost |
| Gleaning / multi-pass per chunk | +3–8 pts recall | 2–3x build cost |
| Human review of top-N nodes | Large, local | Expensive, doesn't scale |

Plot **downstream task success vs. build cost** for 3–4 settings. This is the chart that gets budget approved, because it converts "the graph is expensive" into "the next 5 points of recall cost $18k and here is the success delta."

Track per-build: `extraction_tokens`, `build_time`, `storage_gb`, `$ total`, and `$ per 1k documents`. Rebuild cost after a corpus change is often the number that kills a GraphRAG project in year two.

---

## 7.10 Drift and staleness

The graph is a snapshot. Documents change.

- **Staleness rate**: fraction of nodes whose provenance chunk has been superseded. Target < 5%; above 15% your answers are quietly out of date.
- **Incremental update correctness**: change one document, rebuild incrementally, and diff the graph. Assertions: old edges removed, new edges added, no unrelated churn. Incremental rebuilds are where most production graph bugs live.
- **Deletion handling**: delete a document; are its exclusive nodes and edges removed? Orphaned edges from deleted sources are common and produce uncitable answers.

Add these as deterministic tests in CI, on a tiny fixture corpus. They will catch more production bugs than any judge.

---

## Do this on your system

- [ ] Sample 50 chunks and score self-containment. Fix chunking before anything else.
- [ ] Label 200 entities and 200 relations; compute extraction P/R and hallucinated-triple rate.
- [ ] Build the 150-pair resolution set; compute pairwise P/R; check person vs org thresholds separately.
- [ ] Print the top 20 nodes by degree and eyeball them for over-merging.
- [ ] Run the contradiction detectors (two birth dates, two HQs) as unit tests.
- [ ] Compute provenance link coverage and correctness on 100 edges.
- [ ] Run community detection twice; compare partitions or matched-community Jaccard.
- [ ] Sample 50 summaries; score support rate and coverage.
- [ ] Estimate corpus claim coverage on 50 chunks. Put it next to task success.

## Common mistakes

- **Evaluating only the final answer.** You will spend months tuning prompts against a graph that is missing 40% of its edges.
- **Judging extraction on the first 200 nodes.** They are always your cleanest documents.
- **One resolution threshold for all entity types.** People and organisations need different rules.
- **Ignoring negation and temporality.** These produce confident, wrong, authoritative-sounding answers.
- **No incremental-update tests.** The graph rots silently; nobody notices until a customer does.
- **Optimising modularity.** Optimise stability and downstream success instead.

## Key takeaways

1. The graph is an inferred artifact with an error rate; audit it stage by stage.
2. Measure relation recall on independently annotated source chunks; missing required edges can constrain graph-only reasoning.
3. Entity resolution deserves a labelled pair set and separate person/org thresholds. Bias toward precision.
4. Summary faithfulness is a second hallucination surface; sample by community size.
5. Report corpus coverage beside task success so the ceiling is visible.
6. Incremental rebuild correctness is a deterministic CI test, not an afterthought.

---

**Next:** [Chapter 8 — Graph retrieval evals](08-graph-retrieval-evals.md). A correct graph is necessary, not sufficient. Did the query reach the right part of it?
