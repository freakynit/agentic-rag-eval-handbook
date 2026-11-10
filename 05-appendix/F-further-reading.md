# Appendix F — Further reading

> Reference links checked on 2026-11-10; product features and living standards can change. Small and annotated. Read these for method, not for benchmarks to chase. Every entry is here because it changes a decision you will make, not because it is well cited.

---

## F.1 Read first, if you read three

**"RAG vs. GraphRAG: A Systematic Evaluation and Key Insights"** (2025–2026)
[arxiv.org/abs/2502.11371](https://arxiv.org/abs/2502.11371)
The most useful paper for practitioners because it is mostly a **catalogue of invalid comparisons** and a unified protocol: same preprocessing, same retrieval config, same generation settings. Its findings on cost (construction time, retrieval latency, storage), on sensitivity to construction-model quality, and on position bias in LLM-as-judge summarisation are the three things most likely to invalidate your own ablation. Read §"Evaluation and Efficiency Analyses" and the appendices on cost before you run yours.

**GraphRAG-Bench — "When to use Graphs in RAG: A Comprehensive Analysis..."** (ICLR'26)
[github.com/GraphRAG-Bench/GraphRAG-Benchmark](https://github.com/GraphRAG-Bench/GraphRAG-Benchmark) · [arxiv.org/abs/2506.05690](https://arxiv.org/abs/2506.05690)
Steal the **structure**: it scores graph construction, knowledge retrieval, and final generation separately, across fact retrieval, complex reasoning, contextual summarisation, and creative generation. That four-level task ladder maps cleanly onto Chapter 3's slices. The headline finding is the one that should shape your roadmap: GraphRAG often underperforms vanilla RAG on real-world tasks, and wins only on specific structures. Use their task taxonomy, not their datasets.

**"From Local to Global: A Graph RAG Approach to Query-Focused Summarization"** (Edge et al., Microsoft, 2024)
[arxiv.org/abs/2404.16130](https://arxiv.org/abs/2404.16130)
The origin of community-summary based global search. Read it for the map-reduce decomposition, because that decomposition is exactly why you need **two** faithfulness metrics (map-phase and reduce-phase) rather than one. Most teams report only the final answer's faithfulness and miss reduce-phase drift entirely.

---

## F.2 Evaluation method

**RAGChecker: A Fine-grained Framework for Diagnosing RAG** (2024)
[arxiv.org/abs/2408.08067](https://arxiv.org/abs/2408.08067)
Claim-level entailment scoring with separate retriever and generator diagnostics. A reference for claim-level diagnosis in Chapter 6; the answer-level 2×2 in this handbook is a separate diagnostic view, not a causal attribution method.

**RGB — "Benchmarking Large Language Models in Retrieval-Augmented Generation"** (Chen et al., 2023–2024)
[arxiv.org/abs/2309.01431](https://arxiv.org/abs/2309.01431)
Four capabilities worth stealing as eval slices: noise robustness, negative rejection, information integration, counterfactual robustness. Negative rejection in particular is the metric most teams omit, and the one users notice first.

**ARES: An Automated Evaluation Framework for Retrieval-Augmented Generation Systems** (2023; revised 2024)
[arxiv.org/abs/2311.09476](https://arxiv.org/abs/2311.09476)
The clearest treatment of **judge calibration with prediction-powered inference** — using a small labelled set to correct a judge's bias and produce confidence intervals. Read if you want to report judge-derived metrics with honest error bars.

**"Agentic Retrieval-Augmented Generation: A Survey on Agentic RAG"** (2025–2026)
[arxiv.org/abs/2501.09136](https://arxiv.org/abs/2501.09136)
A taxonomy of agentic RAG architectures by agent cardinality and control structure. Useful for naming what you have built (router, multi-agent, hierarchical) so your eval can be scoped to the right failure modes.

**LangChain — "LLM Evaluation Framework: Trajectories vs. Outputs"**
[langchain.com/resources/llm-evaluation-framework](https://www.langchain.com/resources/llm-evaluation-framework)
Practitioner-written, and the clearest statement of the central point: correct answers hide broken reasoning; score the execution path across grounding, experience quality, and safety. Read in 15 minutes, then read Chapter 9.

**Confident AI / DeepEval — "LLM Agent Evaluation Metrics"**
[confident-ai.com/blog/llm-agent-evaluation-complete-guide](https://www.confident-ai.com/blog/llm-agent-evaluation-complete-guide)
The most complete public list of agent metrics with definitions: task completion, tool correctness, argument correctness, step efficiency, plan adherence, plan quality. Useful as a checklist against Chapter 9; skip the product framing.

**MLflow — "Top Agent Evaluation Frameworks"**
[mlflow.org/top-5-agent-evaluation-frameworks](https://mlflow.org/top-5-agent-evaluation-frameworks)
A vendor comparison, but the **capability axes** it uses are the right ones: custom metrics, human feedback loops, multi-turn support, trace-to-dataset conversion, CI/CD. Use the axes, ignore the rankings.

---

**τ-bench: A Benchmark for Tool-Agent-User Interaction in Real-World Domains** (2024)
[Paper](https://arxiv.org/abs/2406.12045) · [reference implementation](https://github.com/sierra-research/tau-bench)
Introduces pass^k for repeated-run reliability and distinguishes all-k success from pass@k. Use its combinatorial estimator when you run more than k trials per task.

## F.3 Benchmarks worth knowing (for method, not scores)

| Benchmark | Use it for |
|---|---|
| **MultiHop-RAG** | Multi-hop question design; how to construct evidence sets that require joining |
| **FRAMES** | Factuality with multi-hop and multi-constraint questions across documents |
| **STaRK** | Retrieval over structured/semi-structured knowledge (graphs, catalogs) — closer to GraphRAG than text QA |
| **CRAG** | Broad RAG evaluation with question types and entity-popularity stratification |
| **RAGAS testset generator** | Synthetic question generation — read the bias caveats in Chapter 3 first |

Use them to **borrow question shapes and labelling protocols**. Do not adopt their scores as your target; your corpus is the only thing that matters.

---

## F.4 Implementation references

- **OpenTelemetry GenAI semantic conventions** — use supported conventions and pin the version. The former documentation page points to the [GenAI conventions repository](https://github.com/open-telemetry/semantic-conventions-genai). This handbook's compact trace format requires an exporter mapping; it is not the OTel wire schema.
- **RAGAS docs** — the reference definitions for faithfulness, context precision, context recall, noise sensitivity. Check the exact metric variant before comparing scores. Appendix A uses ordinary fixed-budget precision@k; Ragas' Context Precision is rank-sensitive, while ID-based variants have their own definitions. [docs.ragas.io](https://docs.ragas.io)
- **DeepEval docs** — pytest-style agent metrics; the best starting point if you want evals to feel like unit tests. [DeepEval metric documentation](https://deepeval.com/docs/metrics-introduction)
- **Langfuse / Phoenix docs** — OTel-native tracing with annotation queues; read the self-hosting pages if data residency is a constraint.

---

## F.5 What is deliberately not here

- **Leaderboard chasing.** Your system's score on someone else's dataset predicts almost nothing about your product.
- **Prompt-engineering guides.** This handbook assumes the prompt is a variable to be measured, not a craft to be perfected.
- **Vector database comparisons.** Retrieval quality is dominated by chunking, labels, and reranking, not by the index product.
- **"RAG is dead" essays.** Every architecture cycle produces them. The measurement problem stays the same.

---

## F.6 How to read a paper as a practitioner

Three questions, in order:

1. **What did they hold constant, and what did they vary?** If the answer is unclear, the comparison is not usable.
2. **What did they not measure?** Almost always: construction cost, latency tails, abstention, and per-slice results. Those are usually where your decision differs from theirs.
3. **What would have to be true for this result not to apply to me?** Write it down. That list is your ablation design.

If a paper cannot survive those three questions, it is a demo with a PDF.
