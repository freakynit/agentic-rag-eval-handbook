# Appendix B — Schemas and repo layout

> Configuration examples to adapt. These are not JSON Schema or a complete runtime implementation. Model names and trace IDs are illustrative.

---

## B.1 Eval case schema

```yaml
# evals/cases/multi_hop/0042.yaml
id: eval_0042
version: 3
slice: multi_hop                 # single_hop | multi_hop | aggregation | temporal |
                                 # entity_disambiguation | comparative | multi_turn |
                                 # unanswerable | ambiguous | adversarial | stale | long_tail
difficulty: 2                    # 1-3, informational only
source: user                     # user | expert | synthetic
owner: r.okafor
created_at: 2026-11-02
as_of: 2024-12-31T23:59:59Z       # historical business-state snapshot for this case
corpus_snapshot: supplier-audit-2024-12-31
language: en
tenant_scope: [acme]             # null = global

question: "Which of our suppliers also appeared in the 2023 audit findings?"
# For multi-turn:
# turns: ["Show me Q2 revenue.", "And by region?"]

gold_answer: "Northwind Logistics and Calder Freight"

required_facts:
  - "Northwind Logistics"
  - "Calder Freight"
forbidden_facts:
  - "Vantage Supply"             # common distractor

gold_evidence:
  - hop: 1
    doc_id: suppliers-2024.xlsx
    chunk_id: suppliers-2024#c12
    supports: "supplier list"
  - hop: 2
    doc_id: audit-2023.pdf
    chunk_id: audit-2023#c44
    supports: "findings naming Northwind and Calder"

gold_path:                       # graph cases only
  nodes: ["supplier:northwind", "supplier:calder", "audit_finding:AF-2231"]
  edges:
    - {from: "supplier:northwind", to: "audit_finding:AF-2231", type: appears_in}
    - {from: "supplier:calder",    to: "audit_finding:AF-2231", type: appears_in}

gold_entities: []                # this question contains no named-entity mention
# Annotate intermediate mentions separately when scoring the trajectory:
intermediate_gold_entities:
  - {step_index: 4, mention: "Northwind", node_ids: ["supplier:northwind"]}
# "suppliers" and "audit findings" are type/topic mentions, not named entities.

# This is a hybrid-route fixture requiring both evidence channels.
# Task-only cases should accept any valid route instead.
requires_text_evidence: true
requires_graph_evidence: true

path_constraints:
  must_include:
    - {tool: search_docs}
    - {tool: graph_neighbors}
  must_not_include:
    - {tool: web_search}
    - {tool: send_email}
  order_pairs:
    - [search_docs, graph_neighbors]
  max_steps: 10
  max_retrieval_calls: 5
  must_terminate_with: final_answer

must_abstain: false
expected_citations: [audit-2023#c44]
reference_steps: 8

tags: [supplier, audit, multi_hop]
notes: "Distractor: Vantage Supply appears in 2024 list but not audit."
```

### Rules

- `required_facts` and `forbidden_facts` are what make correctness deterministic-ish. Populate both.
- `gold_evidence` at **chunk** granularity, per hop.
- `path_constraints`, not a scripted path. Only require a tool for an explicitly scoped tool/routing contract.
- Record a corpus snapshot and `as_of` timestamp for temporal/current-status questions. Pin chunk IDs to that snapshot; re-chunking requires a label migration.
- For unanswerable cases, use `gold_answer: null`, `required_facts: []`, `gold_evidence: []`, and `must_abstain: true`. Empty labels on an answerable case mean missing annotation, not unanswerability. Clarification cases need `expected_behaviour: clarify` and their own criteria.
- One case per file, one directory per slice. Reviewable diffs matter.

---

## B.2 Trace schema (compact)

```jsonc
{
  "trace_id": "tr_01HX...",
  "case_id": "eval_0042",        // null in prod
  "session_id": "sess_...",
  "turn_index": 0,
  "system": "rag-assistant@1.14.2",
  "commit": "9f3a21c",
  "started_at": "2026-11-10T09:14:02Z",
  "duration_ms": 4210,
  "status": "ok",
  "spans": [
    {
      "span_id": "sp_1", "parent_span_id": null,
      "span_type": "step", "step_index": 0,
      "decision": "decompose", "duration_ms": 380
    },
    {
      "span_id": "sp_2", "parent_span_id": "sp_1",
      "span_type": "retrieval", "step_index": 1,
      "tool_name": "search_docs",
      "query": "2023 audit findings",
      "rewritten_from": "audit findings",
      "retriever_type": "hybrid", "top_k": 8,
      "results": [
        {"doc_id": "audit-2023.pdf", "chunk_id": "audit-2023#c44",
         "score": 0.81, "rank": 1}
      ],
      "duration_ms": 210
    },
    {
      "span_id": "sp_3", "parent_span_id": "sp_1",
      "span_type": "graph_query", "step_index": 2,
      "tool_name": "graph_neighbors",
      "query_entities": [
        {"mentioned_text": "Northwind", "resolved_node_id": "supplier:northwind",
         "link_score": 0.93, "candidates": 4}
      ],
      "expansion_strategy": "k_hop", "hops": 2, "max_hops": 2,
      "nodes": ["supplier:northwind", "audit_finding:AF-2231"],
      "edges": [{"from": "supplier:northwind", "to": "audit_finding:AF-2231",
                 "type": "appears_in"}],
      "duration_ms": 612
    },
    {
      "span_id": "sp_4", "parent_span_id": "sp_1",
      "span_type": "llm_call", "step_index": 3,
      "model": "gpt-x-2026-09", "prompt_version": "synthesize_v14",
      "tokens_in": 3820, "tokens_out": 210, "cost_usd": 0.0121,
      "duration_ms": 1900
    }
  ],
  "outcome": {
    "answer": "...",
    "citations": ["audit-2023#c44"],
    "terminated_reason": "final_answer",   // final_answer | max_steps | abstain | error
    "cost_usd": 0.0189,
    "abstained": false
  }
}
```

The compact example flattens the step tree and omits common fields on child spans. A real trace needs a separate `step` parent per `step_index`, plus common `trace_id`, `span_id`, `parent_span_id`, `started_at`, `duration_ms`, `error`, and `retry_count` fields. Action spans also need `arguments` and `schema_valid`, including rejected calls. `step_index` orders decision cycles; use timestamps and causal links for parallel actions.

This handbook format uses readable IDs. OpenTelemetry exporters must map them to valid SDK-generated trace/span IDs and propagate span context; a label such as `tr_01HX...` is not a valid OTel trace ID.

### Adapter contract for the scoring sketches

The Python sketches assume an adapter that exposes YAML/JSON data as attribute-access objects. It supplies `None` for absent optional attributes and empty lists/dicts for collections. `case.path_constraints` stays a dictionary; `trace.spans_of_type(name)` filters spans. Retrieval results are rank-sorted and de-duplicated, and action wrappers do not duplicate logical tool calls. `normalise(arguments)` returns a stable, hashable representation such as canonical JSON.

Case methods such as `gold_for_subquery(span)`, `args_match(span)`, `success(trace)`, and the Chapter 12 checks require implementation against your labels. Map sub-queries to labelled evidence needs, not exact query strings. `success` is the Chapter 10 product scorer; it must not call `score_trajectory` recursively. Unknown/missing annotations route to review. The sketches are not directly executable against the YAML examples without this adapter.

### Span type enum

`step` · `llm_call` · `retrieval` · `graph_query` · `tool_call` · `rerank` · `filter` · `guardrail` (the trace envelope is the run)

### Required minimum per type

| Type | Required fields beyond the common set |
|---|---|
| `retrieval` | `tool_name`, `query`, `top_k`, `results[].chunk_id`, `.rank`, `.score` |
| `graph_query` | `tool_name`, `query_entities[].resolved_node_id`, `.link_score`, `hops`, `nodes`, `edges` |
| `llm_call` | `model`, `prompt_version`, `tokens_in`, `tokens_out`, `cost_usd` |
| `tool_call` | `tool_name`, `arguments`, `schema_valid` |

Every action span should expose a logical `tool_name` and a normalised `arguments` object. For `retrieval` and `graph_query`, `arguments` can be materialised from `query` / `query_entities` and the tool parameters; the point is that trajectory scoring sees a comparable object, not a free-text blob.

---

## B.3 Failure label schema

```yaml
case_id: eval_0042
trace_id: tr_01HX...
primary_bucket: graph_reach_miss   # see Ch.12 for the enum
secondary_buckets: [entity_link_miss]
severity: high                     # low | med | high | critical
expected: "Northwind and Calder"
actual: "Northwind only"
user_impact: "wrong supplier named; would have triggered an unnecessary refund"
fix_hypothesis: "max_hops=1 for entity queries; use path search or raise to 2"
labelled_by: r.okafor
labelled_at: 2026-11-10
model_versions:
  generator: gpt-x-2026-09
  judge: judge-y
  graph_build: 2026-10-28
```

---

## B.4 Gates file

```yaml
# evals/gates.yaml
version: 4
judge: {model: judge-y, prompt: faithfulness_v7, calibrated: 2026-10-02}
eval_set: frozen_v3

absolute:
  task_success:        {min: 0.70}
  claim_support:       {min: 0.90}
  citation_support:    {min: 0.80}
  acl_violations:      {max: 0}
  injection_successes: {max: 0}
  p95_latency_ms:      {max: 6000}
  cost_per_success:    {max: 0.06}

relative:                 # paired comparison vs main, same batch
  task_success:        {max_drop: 0.03}
  pass3:               {max_drop: 0.05}
  evidence_recall:     {max_drop: 0.04}
  edge_recall:         {max_drop: 0.05}
  slice_success:       {max_drop: 0.08, min_slice_n: 50}  # below 50: report, don't gate

flake_policy:
  scheduled_runs_per_strict_case: 3  # run all cases, including first-attempt passes
  strict_case_any_failure: block_and_triage
  mixed_outcome: add_to_flip_set_and_open_ticket
  hard_zero_violation: block_immediately
  diagnostic_reruns_replace_original: false
```

---

## B.5 Repo layout

```
evals/
├── README.md
├── manifest.yaml                  # per-run: commit, models, judge, index build
├── gates.yaml
├── cases/
│   ├── single_hop/*.yaml
│   ├── multi_hop/*.yaml
│   ├── aggregation/*.yaml
│   ├── temporal/*.yaml
│   ├── entity_disambiguation/*.yaml
│   ├── comparative/*.yaml
│   ├── multi_turn/*.yaml
│   ├── unanswerable/*.yaml
│   ├── ambiguous/*.yaml
│   ├── adversarial/*.yaml
│   ├── stale/*.yaml
│   └── long_tail/*.yaml
├── sets/
│   ├── smoke.txt                  # 40-60 case ids
│   ├── frozen_v3.txt
│   ├── core30.txt                 # pass^k set
│   └── judge_calibration.txt
├── scorers/
│   ├── retrieval.py
│   ├── graph.py
│   ├── generation.py
│   ├── trajectory.py
│   └── attribution.py             # 9-bucket classifier
├── judges/
│   ├── faithfulness_v7.md
│   ├── correctness_v3.md
│   ├── trajectory_v2.md
│   └── graph_relevance_v1.md
├── graph_audit/
│   ├── entities_200.yaml
│   ├── relations_200.yaml
│   ├── resolution_pairs_150.yaml
│   └── report.md
├── runs/
│   └── 2026-11-10_nightly/
│       ├── manifest.yaml
│       ├── report.md
│       ├── per_case.csv
│       └── slices.csv
└── tests/                          # T0 contract tests
    ├── test_span_contract.py
    ├── test_acl.py
    └── test_incremental_rebuild.py
```

---

## B.6 Report template

```markdown
# Eval report — 2026-11-10 nightly

manifest: rag-assistant@1.14.2 (9f3a21c) · judge-y/faithfulness_v7 · frozen_v3

## Headline
task_success 0.81 (n=200, 95% CI 0.75–0.87)
baseline    0.78 → +3.0 pts; paired Δ 95% CI −1.0 to +7.0 pts (inconclusive)
cost/success $0.041 · p95 4.2s · pass^3 0.81

## Slices
| slice | n | success | Δ | verdict |
|---|---|---|---|---|
| single_hop | 80 | 0.91 | +1 | flat |
| multi_hop | 50 | 0.79 | +6 | directional gain; paired CI needed |
| aggregation | 30 | 0.68 | −4 | watch |
| temporal | 22 | 0.55 | −12 | investigate; small n |
| unanswerable | 18 | 0.94 | +2 | flat |

## Failure mix
retrieval_miss 16 (▲6) · entity_link_miss 9 (▲3) · reasoning_error 7 (=)
process_error 4 (▼2) · unclassified 2

## Notes
Scores are illustrative rounded means over repeated runs; record runs per case.
Temporal failures traced to eval_0071–0079: date filter dropped in query rewrite.
Owner: @retrieval. Fix in PR #812.
```
