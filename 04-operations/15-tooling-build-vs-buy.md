# Chapter 15 — Tooling: build vs buy

> **Chapter 15** · Part IV: Operating it
> **Time:** 15 minutes
> **You will leave with:** a category map, an honest capability comparison, and a decision rule for what to install versus what to write.

---

## 15.1 Buy the plumbing, write the semantics

The split that works consistently:

| Buy | Build |
|---|---|
| Trace storage, UI, search, replay | Your span schema and what counts as a "step" |
| Dataset/version management | The eval cases themselves |
| Annotation queues, reviewer UI | Your rubric and severity model |
| Standard judge metrics (faithfulness, relevancy) | Your domain rubrics (claim support against *your* compliance rules) |
| Run orchestration, parallelism, caching | Graph-specific metrics (path recall, link accuracy) |
| Dashboards, comparison views | Gate thresholds and slice policy |

Everything in the right column is differentiated. Everything in the left is commodity, and building it yourself costs two engineers a quarter for a worse UI.

Corollary: **do not write your own tracing backend.** Instrument with OpenTelemetry, emit standard spans, and point them at any vendor. That keeps the door open and costs you an afternoon.

---

## 15.2 Categories and representative options

Representative options, checked against linked product documentation on 2026-11-10. This is a capability map, not a ranking. Validate deployment, pricing, retention, and export requirements for the edition you would use.

### Tracing and observability

| Tool | Shape | Strengths | Watch for |
|---|---|---|---|
| **[LangSmith](https://docs.langchain.com/langsmith/observability)** | Managed platform; check deployment options | Tracing, evaluations, datasets, and framework integrations | Test trace export, deployment requirements, and usage-based costs |
| **[Langfuse](https://langfuse.com/docs)** | Open source + cloud | Tracing, cost analysis, prompt management, and evaluations | Budget self-hosting operations; verify required edition features |
| **[Arize Phoenix](https://arize.com/docs/phoenix)** | Open source + hosted options | OTel-based tracing, evaluations, datasets, and experiments | Test your graph payloads and annotation workflow |
| **[Braintrust](https://docs.braintrust.dev/)** | Managed platform; check deployment options | Experiments, tracing, online scoring, feedback, and alerts | Validate production sampling, retention, and export requirements |
| **[W&B Weave](https://docs.wandb.ai/weave)** | W&B platform | Agent tracing, evaluations, versioning, and monitoring | Check deployment and storage requirements |
| **[MLflow](https://mlflow.org/docs/latest/genai/)** | Open source + managed offerings | Tracing, evaluations, monitoring, and version tracking | Managed and self-hosted features can differ |
| **Grafana / Datadog + OTel** | Existing infrastructure stack | Reuse operational telemetry for latency, errors, and cost attributes | Validate LLM-specific payload handling, judging, and annotation needs |

### Metric libraries

| Library | Best at | Notes |
|---|---|---|
| **[Ragas](https://docs.ragas.io)** | Classic RAG metrics: faithfulness, answer relevancy, context precision/recall, noise sensitivity | Check each metric variant and its label requirements; names may differ from this handbook's formulas |
| **[DeepEval](https://deepeval.com/docs/metrics-introduction)** | Agent metrics in a pytest style: task completion, tool correctness, argument correctness, step efficiency, plan adherence/quality, G-Eval | Best fit if you want evals to feel like unit tests in CI |
| **TruLens** | RAG triad (context relevance, groundedness, answer relevance), feedback functions | Good conceptual model; lighter on trajectories |
| **Phoenix evals** | Configurable LLM evaluators and dataset/trace evaluation workflows | Pairs naturally with Phoenix tracing |
| **promptfoo** | CLI-first, YAML test suites, CI, red-teaming | Excellent for smoke gates and prompt regression; less for trajectory analysis |
| **Custom** | Graph metrics, domain rubrics, conditional analyses | Unavoidable; see Appendix B for the shape |

### Supporting casts

- **Synthetic data:** RAGAS testset generation, promptfoo, or a purpose-built generator. Read Chapter 3 warnings first; synthetic questions are systematically too clean.
- **Human labelling:** the annotation queue in whichever tracing tool you bought; Label Studio or Argilla if you need a standalone.
- **Graph benchmarks:** **GraphRAG-Bench** (ICLR'26) for methodology — it scores graph construction, retrieval, and generation separately across fact retrieval, complex reasoning, contextual summarisation, and creative generation. Use its *structure*, not its datasets; your corpus is the point.
- **Comparisons:** the "RAG vs. GraphRAG" unified-protocol work is worth reading for one reason: it is a checklist of the ways a comparison can be invalid (different token budgets, position bias in pairwise judging, unmeasured construction cost).

---

## 15.3 Decision rules

**Choose a tracing vendor when:** you have >2 people working on the system, or >1 deployable, or you need replay. Before that, a JSONL file and a notebook are fine and faster.

**Choose OTel-native when:** you already run Grafana/Datadog, or you are in a regulated environment that will not accept a new SaaS data processor, or you expect to change vendors.

**Choose a metric library when:** you want the standard RAG metrics without re-deriving the prompts. Do not expect any library to give you trajectory metrics that match your agent's step model — you will write the adapters.

**Write your own when:** the metric is graph-specific, domain-specific, or conditional. In practice you will write 60–70% of the metrics in this handbook. That is normal and not a sign of tooling failure.

**Never buy:** a "quality score" product that gives you one number and no trace. It cannot help you fix anything.

---

## 15.4 Reference stacks

### Small team (1–3 engineers, one product)
```
Tracing:   Langfuse (self-hosted) or LangSmith
Metrics:   DeepEval for agent, RAGAS for retrieval
Data:      YAML cases in-repo + the tool's dataset feature
CI:        promptfoo or DeepEval in GitHub Actions
Judge:    one strong hosted model, pinned, self-hosted if data rules require
Graph:    custom scorers (Ch. 7–8)
```

### Mid-size (4–10 engineers, multiple surfaces)
```
Tracing:   LangSmith or Phoenix + OTel collector
Metrics:   library metrics + a shared internal `evals/` package
Data:      versioned datasets, frozen/working/holdout split
CI:        T0 contract tests + T1 smoke on PR, T2 nightly
Judge:    cascade, cached, batch-priced
Graph:    dedicated graph health report (Ch. 7)
```

### Enterprise / regulated
```
Tracing:   OTel + self-hosted backend (Langfuse/Phoenix/MLflow), regional
Metrics:   internal package, versioned
Judge:    self-hosted for customer data, hosted only for synthetic
Data:      no eval data leaves the boundary; retention tiers enforced
CI:        gates in the internal pipeline; ACL and injection as hard zeros
Graph:    construction audit as a scheduled job, not an ad-hoc script
```

---

## 15.5 Integration details that save a week

- **Use OpenTelemetry with supported GenAI conventions** and pin the instrumentation/convention version. See the [GenAI conventions repository](https://github.com/open-telemetry/semantic-conventions-genai). OTel transport does not guarantee identical attribute interpretation across vendors; test an export/import round trip with your payloads.
- **Propagate the active OTel span context** across framework and service boundaries. Let the SDK create valid IDs; attach an application request ID separately. Independent root traces from each library break causal joins.
- **Put `case_id` on eval runs.** This one field is the join between your dataset and your traces; without it every cross-analysis is manual.
- **Store large I/O out of band**, keyed by content hash. Trace stores bill by volume.
- **Version everything**: prompt, judge, eval set, model, retriever config. The report should be reproducible from a single manifest.

```yaml
# manifest.yaml — commit this next to the report
system: rag-assistant@1.14.2
commit: 9f3a21c
model: gpt-x-2026-09               # placeholder; use an actual pinned model ID
model_settings: {temperature: 0, max_output_tokens: 2000}
otel_semconv_version: "<pinned supported version>"
cache_policy: cold_per_run
as_of: 2026-11-10T00:00:00Z
judge: {model: judge-y, prompt: faithfulness_v7, calibrated: 2026-10-02}
eval_set: frozen_v3
retriever: {index: prod-2026-11-01, k: 8, rerank: true}
graph: {build: 2026-10-28, hops: 2}
```

---

## Do this on your system

- [ ] List your current tools by category. Mark anything you built that is in the "buy" column.
- [ ] Add OTel-style span attributes if you are not already; it costs a day and de-risks the next two years.
- [ ] Pick one metric library and port three metrics to it. Do not port all of them.
- [ ] Move eval cases into version control with a manifest.
- [ ] Decide the judge hosting question with security, in writing.
- [ ] Delete any tool that produces a number you have never acted on.

## Common mistakes

- **Building a tracing UI.** Two engineers, one quarter, worse than off-the-shelf.
- **Buying three overlapping tools.** Pick one tracing backend; libraries can coexist.
- **Letting each library mint trace IDs.** You lose the ability to join.
- **No manifest.** Reports become unreproducible and comparisons become folklore.
- **Assuming a library has your graph metrics.** None do; plan the custom work.

## Key takeaways

1. Buy tracing, datasets, annotation, and standard judges. Build semantics, domain rubrics, and graph metrics.
2. Instrument with OTel so the vendor decision stays reversible.
3. One tracing backend, libraries for metrics, custom code for everything conditional.
4. `case_id` on every eval trace is the single most valuable integration detail.
5. Version prompts, judges, datasets, models, and index builds in a manifest.

---

**Next:** [Chapter 16 — Rollout: the first 90 days](16-rollout-90-days.md). How to actually stand this up with a team.
