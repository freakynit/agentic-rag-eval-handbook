# Chapter 4 — Tracing and instrumentation

> **Chapter 4** · Part I: Foundations
> **Time:** 20 minutes
> **You will leave with:** a span schema, a sampling policy, and the trace→eval loop that turns production failures into regression tests.

---

## 4.1 Instrumentation precedes evaluation

Every trajectory metric in Part III is a function of your trace. If your system logs only `(question, answer, latency)`, roughly half of this handbook is unavailable to you, and no amount of metric tooling fixes it.

Adopt this rule: **any step that could be the reason an answer is wrong must emit a span.** That includes the ones you think are boring — chunk ranking, graph expansion, prompt assembly, rerank.

Instrument once, at the framework boundary, not per-call-site. If you use LangGraph, LlamaIndex, Haystack, DSPy, or a custom loop, wrap the executor and emit spans from there.

---

## 4.2 The span model

Use a run → step → action hierarchy. Map it to your backend explicitly; the compact JSON below is a handbook format, not an OpenTelemetry wire format.

```
run (trace)                    one user request, one agent execution
├── step                       one decision → action cycle
│   ├── llm_call               planning, routing, synthesis
│   ├── retrieval              vector / keyword / hybrid
│   ├── graph_query            entity link, neighbors, path, community
│   ├── tool_call              any external tool
│   └── rerank / filter        post-processing
└── step ...
```

### Required fields

| Field | On | Why |
|---|---|---|
| `trace_id`, `span_id`, `parent_span_id` | all | Joining, tree rendering |
| `case_id` | run | Link to the eval case (null for prod) |
| `step_index` | step + action spans | Ordering, step efficiency |
| `span_type` | all | Dispatch to the right scorer |
| `started_at`, `duration_ms` | all | Latency attribution |
| `tokens_in`, `tokens_out`, `cost_usd` | llm_call | Cost attribution per step |
| `model`, `prompt_version` | llm_call | Model/prompt regression isolation |
| `input`, `output` | all | Replay and judging |
| `error`, `retry_count` | all | Failure taxonomy |
| **Retrieval** | | |
| `query` (post-rewrite) | retrieval | Per-sub-query recall |
| `top_k`, `retriever_type` | retrieval | Reproducibility |
| `results[].doc_id`, `.chunk_id`, `.score`, `.rank` | retrieval | Recall / precision / nDCG |
| **Graph** | | |
| `query_entities[]` with `mentioned_text`, `resolved_node_id`, `link_score` | graph_query | Entity-link accuracy |
| `expansion_strategy`, `hops`, `max_hops` | graph_query | Hop analysis |
| `nodes[]`, `edges[]` (ids + types) | graph_query | Path recall, subgraph precision |
| `community_ids[]` | graph_query | Global search evaluation |
| **Action** | | |
| `tool_name`, `arguments` (normalised), `raw_arguments` on rejected calls | retrieval, graph_query, tool_call | Tool + argument correctness; retain rejected attempts so invalid calls remain measurable |
| `schema_valid`, `validation_error` | every attempted action | Invalid-call rate, including calls rejected before execution |

`span_type` says what kind of action it was; `tool_name` says which logical tool the agent chose (`search_docs`, `graph_neighbors`, …). Trajectory scoring needs both, which is why action spans carry both fields.

Two fields are worth more than the rest: **`chunk_id` on every retrieval result** and **`resolved_node_id` plus `link_score` on every entity mention**. Without them you cannot compute evidence recall or entity-link accuracy, which are the two highest-yield component metrics in the whole handbook.

---

## 4.3 A minimal schema

```jsonc
{
  "trace_id": "tr_01HX...",
  "case_id": "eval_0042",           // null in prod
  "session_id": "sess_...",          // for multi-turn
  "turn_index": 2,
  "started_at": "2026-11-10T09:14:02Z",
  "duration_ms": 4210,
  "status": "ok",                    // ok | error | timeout | aborted
  "spans": [
    {
      "span_id": "sp_1",
      "parent_span_id": null,
      "span_type": "step",
      "step_index": 0,
      "decision": "decompose",
      "duration_ms": 380
    },
    {
      "span_id": "sp_2",
      "parent_span_id": "sp_1",
      "span_type": "graph_query",
      "tool_name": "graph_neighbors",
      "step_index": 1,
      "query_entities": [
        {"mentioned_text": "Northwind", "resolved_node_id": "supplier:northwind",
         "link_score": 0.93, "candidates": 4}
      ],
      "expansion_strategy": "k_hop",
      "hops": 2,
      "max_hops": 2,
      "nodes": ["supplier:northwind", "audit_finding:AF-2231"],
      "edges": [{"from": "supplier:northwind", "to": "audit_finding:AF-2231",
                 "type": "appears_in"}],
      "duration_ms": 612
    },
    {
      "span_id": "sp_3",
      "parent_span_id": "sp_1",
      "span_type": "llm_call",
      "step_index": 2,
      "model": "gpt-x-2026-09",
      "prompt_version": "synthesize_v14",
      "tokens_in": 3820, "tokens_out": 210, "cost_usd": 0.0121,
      "input_hash": "sha256:...",     // store full input separately if large
      "output": "...",
      "duration_ms": 1900
    }
  ],
  "outcome": {
    "answer": "...",
    "citations": ["audit-2023#c44"],
    "terminated_reason": "final_answer"   // final_answer | max_steps | error | abstain
  }
}
```

Store large payloads out of band, keyed by hash and an access-controlled URI. A hash verifies identity but cannot reconstruct a prompt; replay also needs the payload, corpus/index and graph snapshots, model settings, tool results, and configuration versions. Redact before storage and scope payload access to the tenant.

The example flattens the step tree for brevity: `sp_2` and `sp_3` share the planning step as parent. In a real trace, each `step_index` gets its own `step` parent; `step_index` is the ordering key, and the parent tree is for causal drill-down.

---

## 4.4 Sampling policy

| Context | Sampling | Rationale |
|---|---|---|
| Eval runs | 100% | You need every run for pass^k |
| Production errors / escalations | 100% | These become your next eval cases |
| Production traffic | 5–20% | Enough for slice metrics; cost-controlled |
| High-value tenants / regulated flows | 100% | Audit and dispute resolution |
| Canary | 100% | You are explicitly comparing |

Always sample **whole traces**. To retain all errors, use tail sampling or buffering: head sampling cannot know which requests will later fail. Keep a separate uniform sample for production-rate estimates; mixing oversampled errors with random traces biases those estimates unless you weight by inclusion probability.

---

## 4.5 Privacy and retention

Non-negotiable if you are in a professional setting:

- **Redact at emit time**, not at read time. PII in the trace store is a breach waiting to happen.
- Hash or drop raw prompts that may contain customer data; keep a pointer to a redacted store with tighter access.
- **Example retention tiers:** tightly restricted raw payloads up to 30 days, redacted traces 90 days, aggregates under an explicit retention policy. Adjust to your data rules; aggregates can still contain sensitive small-group information.
- Respect data-residency: trace stores are a common accidental cross-border transfer.
- Document what the judge sees. If your judge is a hosted API, retrieved customer documents are leaving your boundary — many enterprises forbid this. Plan for a local judge (see Chapter 13).

---

## 4.6 The trace → eval loop

This is the mechanism that keeps an eval set relevant. Build it once; it pays forever.

```
production traffic
      │
      ├─▶ automatic scorers (cheap, sampled)
      │        │
      │        └─▶ low score? ──▶ annotation queue
      │
      └─▶ user signal (thumbs, escalation, edit, abandon, retry)
               │
               └─▶ negative? ──▶ annotation queue

annotation queue ─▶ human labels ─▶ new eval case (with evidence) ─▶ regression suite
```

Four design details that make the difference:

1. **Deduplicate.** Cluster failing traces by (failure type, intent) so you get 40 distinct cases, not 400 near-duplicates.
2. **Capture the trace.** Every new eval case links to the original `trace_id`. When it regresses later you can compare against the exact historical run.
3. **Label before fixing.** The case must exist in a red state. Fixing first and writing the test after is how you get a test that asserts the bug.
4. **Close the loop publicly.** Post a weekly "new cases added / cases turned green" number. This is what makes the program stick.

---

## 4.7 What to build first, in order

If you are starting from zero logging:

1. `trace_id`, `step_index`, `span_type`, `duration_ms`, `error` on every step. (A day.)
2. `chunk_id` + `rank` on every retrieval result. (Half a day, unlocks Chapter 5.)
3. `resolved_node_id` + `link_score` on every entity mention. (One to three days, unlocks Chapter 8.)
4. Token and cost per LLM call. (Half a day, unlocks Chapter 11.)
5. Full I/O per span, redacted. (One day, unlocks replay and Chapter 12.)
6. The annotation queue + case promotion. (One to two weeks, unlocks everything long-term.)

Do not buy a tracing vendor before step 1 exists. You will pay for a UI that displays three fields.

---

## Do this on your system

- [ ] Take one real run and write out its spans on paper. Mark every span you currently do not emit.
- [ ] Add `chunk_id` + `rank` to retrieval output today. It is the cheapest high-value change in this handbook.
- [ ] Add `resolved_node_id` + `link_score` + `candidates` to entity linking.
- [ ] Define your sampling policy in one table and implement it at the emitter, not the reader.
- [ ] Stand up the annotation queue, even if it is a spreadsheet and a Slack workflow. Automate later.
- [ ] Confirm with legal/security where traces may live and whether a hosted judge is permitted.

## Common mistakes

- **Logging the final prompt only.** You cannot separate prompt-assembly bugs from model bugs.
- **Sampling individual spans.** Partial traces mislead every trajectory metric.
- **No `case_id` linkage.** Eval runs and prod runs live in separate worlds and you cannot compare them.
- **Storing untruncated full documents in the trace store.** Costs balloon; retention becomes a liability.
- **Instrumenting call sites individually.** Someone adds a retriever and forgets. Wrap the executor.

## Key takeaways

1. Any step that could cause a wrong answer needs a span.
2. The two highest-value fields are `chunk_id` on retrieval results and `resolved_node_id` + `link_score` on entity mentions.
3. Sample whole traces; 100% on eval runs and errors.
4. Redact at emit, set retention tiers, and check whether your judge is allowed to see customer data.
5. The trace→annotation→regression loop is what keeps the eval set alive. Build it early.

---

**Next:** [Chapter 5 — Retrieval evals](../02-component-evals/05-retrieval-evals.md). The first component, and the one that causes most failures.
