# The Agentic RAG & GraphRAG Evaluation Handbook

A practitioner's guide to measuring, debugging, and shipping retrieval-augmented systems that plan, call tools, and traverse graphs.

**Audience:** engineers, ML/data scientists, and technical product leads already building (or about to build) an agentic RAG or GraphRAG product. No research background required. Assumes you know what a vector index, a tool call, and a knowledge graph are.

**What this is:** a working manual. Every chapter ends with an artifact to produce and a checklist you can run against your own system.

**What this is not:** a survey of academic benchmarks, a vendor comparison, or a theory text. Benchmarks are referenced only where they give you a reusable method.

---

## The thesis in four lines

1. A single output score cannot tell you *where* an agentic RAG system broke. You need evaluation at every seam: evidence, retrieval step, graph structure, trajectory, outcome.
2. GraphRAG adds a new class of failures — a graph that is wrong, a graph that is right but unused, and a graph that works but costs 20x more than vector RAG. None of these are visible in answer quality alone.
3. Quality numbers are meaningless without cost, latency, and reliability (pass^k) reported next to them.
4. The eval set, not the metric library, is the asset. Curated questions with gold evidence and expected tool paths are what make every other measurement possible.

---

## Handbook map

### Part I — Foundations

| # | Chapter | Question it answers | Read |
|---|---|---|---|
| 1 | [Why evals break](00-foundations/01-why-evals-break.md) | Why does my input→output test suite miss the bugs users hit? | 20 min |
| 2 | [The eval stack](00-foundations/02-the-eval-stack.md) | What do I measure, at which layer, and how do I attribute a failure? | 25 min |
| 3 | [Eval dataset design](01-datasets-and-traces/03-eval-dataset-design.md) | What questions do I need, and what do I label? | 30 min |
| 4 | [Tracing and instrumentation](01-datasets-and-traces/04-tracing-and-instrumentation.md) | What must I record to score a 40-step run? | 20 min |

### Part II — Component evals

| # | Chapter | Question it answers | Read |
|---|---|---|---|
| 5 | [Retrieval evals](02-component-evals/05-retrieval-evals.md) | Did we fetch the evidence, for every sub-query? | 25 min |
| 6 | [Generation evals](02-component-evals/06-generation-evals.md) | Is every claim supported, and is the answer actually right? | 25 min |
| 7 | [Graph construction evals](02-component-evals/07-graph-construction-evals.md) | Is the graph itself correct and complete enough to trust? | 30 min |
| 8 | [Graph retrieval evals](02-component-evals/08-graph-retrieval-evals.md) | Does the retrieved subgraph contain the reasoning path? | 30 min |

### Part III — System evals

| # | Chapter | Question it answers | Read |
|---|---|---|---|
| 9 | [Trajectory evals](03-system-evals/09-trajectory-evals.md) | Was the path sound, efficient, and correctly terminated? | 30 min |
| 10 | [End-to-end, reliability, online](03-system-evals/10-end-to-end-and-online.md) | Did the task succeed, reproducibly, in production? | 25 min |
| 11 | [Cost, latency, and is the graph worth it](03-system-evals/11-cost-and-graph-ablation.md) | Are we buying quality we could not get cheaper? | 20 min |
| 12 | [Failure taxonomy](03-system-evals/12-failure-taxonomy.md) | Which of the nine primary buckets actually broke? | 20 min |

### Part IV — Operating it

| # | Chapter | Question it answers | Read |
|---|---|---|---|
| 13 | [LLM-as-judge operations](04-operations/13-llm-as-judge-operations.md) | How do I trust an automated grader? | 25 min |
| 14 | [CI gates and release process](04-operations/14-ci-gates-and-release.md) | What blocks a deploy? | 20 min |
| 15 | [Tooling: build vs buy](04-operations/15-tooling-build-vs-buy.md) | What do I actually install? | 15 min |
| 16 | [Rollout: the first 90 days](04-operations/16-rollout-90-days.md) | How do I stand this up with a real team? | 20 min |

### Appendices

| | Appendix | Use |
|---|---|---|
| A | [Metric reference](05-appendix/A-metric-reference.md) | Lookup table: definition, formula, data needed, target |
| B | [Schemas and repo layout](05-appendix/B-schemas-and-repo-layout.md) | Copy-paste YAML/JSON for eval cases and spans |
| C | [Judge prompt templates](05-appendix/C-judge-prompts.md) | Four grader templates to adapt and calibrate |
| D | [Worked example](05-appendix/D-worked-example.md) | One question traced through all layers with numbers |
| E | [Anti-patterns](05-appendix/E-anti-patterns.md) | Fifteen ways teams waste a quarter on evals |
| F | [Further reading](05-appendix/F-further-reading.md) | Annotated, small, current |

---

## Reading paths

**You have one day and need a plan by Friday.**
1 → 2 → 3 → 5 → 6 → 9 → 11 → 14. Skim Appendix A. Skip the graph chapters unless the graph is already in production.

**You own the graph pipeline.**
7 → 8 → 11 → 12 → 3. Then 5 for the vector baseline you must beat.

**You are the engineering lead standing this up.**
All of Part I, then 9, 10, 13, 14, 15, 16.

**You are the PM or technical founder.**
1, 2, 11, 12, 16, plus Appendix D. You need the vocabulary and the cost argument, not the formulas.

**You are debugging a specific regression today.**
12 → 4 → the component chapter that matches the failure bucket.

---

## Conventions

- **`k` in retrieval metrics** means a fixed retrieval budget. In `pass^k` and `pass@k`, it means the number of trials. State the retrieval stage (candidate fetch or final context) and its budget with every recall number.
- **Thresholds and cost/quality ranges** are illustrative planning heuristics, not validated industry norms. Calibrate them to your task, risk, traffic, and measurement uncertainty. Numerical scenarios are teaching examples unless a source is cited.
- **`pass^k`** means k independent runs of the same task, all of which must succeed. Not the same as pass@k.
- Python blocks are scoring sketches to adapt, not an executable package. YAML and JSONC blocks are configuration examples, not formal schemas. Names such as `gpt-x-2026-09` and `judge-y` are placeholders. Appendix B explains the adapter contract.
- **Slice** means a stratum of the eval set (question type, tenant, language, document age). Almost every metric in this handbook is reported per slice, never only in aggregate.

---

## The eight artifacts you should finish with

Work the chapters in order and you end up with these. They are the deliverable of the whole exercise.

1. A **frozen eval set** of 150–300 cases, stratified, with gold evidence and acceptable tool paths.
2. A **trace schema** and instrumentation covering every retrieval, graph query, LLM call, and tool call.
3. A **retrieval report** per slice: evidence recall, context precision, retrieval efficiency.
4. A **generation report** per slice: claim support rate, correctness, completeness, false-answer / over-rejection pair.
5. A **graph health report**: extraction F1, resolution quality, coverage, community-summary faithfulness.
6. A **trajectory report**: tool correctness, step efficiency, termination, pass^3.
7. A **cost/ablation table** proving the graph (or agent loop) earns its keep.
8. A **CI gate** with per-slice floors and a regression policy, wired into your release process.

---

## A note on honesty

Two failure modes kill eval programs, and both are social rather than technical:

- **The vanity dashboard.** A single "quality score" that goes up when the team does well and is never used to block anything. Verify that gates can block a deliberately seeded regression; a healthy program need not have blocked a real release yet.
- **The perfect suite that ships nothing.** A 4,000-case benchmark nobody runs because it takes six hours. Fast smoke evals that run on every PR beat a beautiful suite that runs quarterly.

Chapter 14 is about avoiding both.

---

*Start with [Chapter 1](00-foundations/01-why-evals-break.md).*
