# Chapter 1 — Why evals break when retrieval starts making decisions

> **Chapter 1** · Part I: Foundations
> **Time:** 20 minutes
> **You will leave with:** a mental model of the three evaluation regimes, and a list of the specific failures your current test suite is structurally blind to.

---

## 1.1 The assumption your eval suite was built on

Almost every RAG eval setup inherited the same shape:

```
question ──▶ [pipeline] ──▶ answer
                               └── compare to gold answer, score, done
```

That shape encodes a hidden assumption: **one question produces one retrieval pass and one answer.** Everything downstream — recall@k, faithfulness, answer similarity — scores that single hop.

Agentic RAG violates it in three ways at once:

- The agent **decomposes** one question into several sub-queries, some of which it invents mid-run.
- The agent **decides** whether to retrieve at all, which retriever to use, and when to stop.
- The agent **acts** on intermediate results: re-plans, abandons a branch, escalates to a different tool.

GraphRAG violates it in a fourth way: part of the system's knowledge may now be a **model output** (an extracted graph), built once, silently, and reused for every query. It can be wrong in ways no answer-level metric will ever reveal.

---

## 1.2 Five failures output-only evaluation cannot see

These are not edge cases. In practice they are most of your production incidents.

### 1. Hidden failures behind correct answers
The agent retrieves an irrelevant document, the synthesis model gets lucky (or already knew the fact), and the answer is right. Your eval passes. Next month the same path fails on a question where the model *doesn't* know the fact — and you have no signal, because you never scored the retrieval. **A correct answer is not evidence of a correct process.**

### 2. Stochastic trajectories
The same input can produce a 3-step run and a 22-step run. A single-run pass rate hides this completely. An agent with pass@1 ≈ 0.85 and pass^3 ≈ 0.61 is reliable on only 61% of cases across three runs; the other 39% have at least one failure. You need reliability, not accuracy.

### 3. Error compounding
If every step is necessary and each has 97% conditional success given prior success, ten steps yield about 74% end-to-end success (0.97^10). A marginal step-accuracy average alone does not justify this multiplication. Output-only evals report the 74% and give you nothing about which step is the leak. Worse, agent failures are correlated: a bad entity link poisons every subsequent hop.

### 4. Cost and latency as first-class product properties
An agent that improves faithfulness by 4 points and costs 6x per query is a regression for most businesses. This never appears in an answer-quality score, so it never appears in a release decision.

### 5. Multi-turn state
Turn 4 depends on a constraint stated in turn 2. Output-only evals score each turn in isolation and cannot see dropped constraints, re-fetching, or drift.

---

## 1.3 What changes when you add a graph

Graph-based retrieval adds construction and traversal failure modes. Stale indexes, unused retrieval, and poor cost trade-offs also occur in vector RAG; the table shows their graph-specific forms.

| Failure class | What it looks like | Why answer-level evals miss it |
|---|---|---|
| **The graph is wrong** | Two distinct people merged into one node; a relation extracted with the wrong direction; a hallucinated triple | The answer can still be plausible, especially for single-hop questions that never touch the bad region |
| **The graph is right but unused** | Entity linking sends the query to the wrong node; the traversal stops before the second hop; the agent answers from the vector index anyway | Answer quality looks identical to a system that never had a graph |
| **The graph works but does not pay** | Quality improves 3 points on 8% of queries while indexing cost triples | You only see the win, because nobody reports the denominator |

There is a fourth, subtler one: **global and local search fail differently.** Community-summary ("global") answers can be fluent, well-structured, and unfaithful to the underlying nodes. Entity-centric ("local") answers can be faithful but incomplete because the traversal missed a hop. If you report one blended score, these cancel each other out and you learn nothing.

> **Rule of thumb:** if you cannot answer "which component failed?" from your eval report, you have a benchmark, not an eval system.

---

## 1.4 The three regimes

Stop thinking about "RAG eval" as one thing. There are three distinct regimes with different metrics, different data requirements, and different cost profiles.

### Regime A — Component evals
Score a single stage in isolation, with fixed inputs.

- Retrieval given a sub-query
- Generation given a fixed context
- Entity extraction given a fixed chunk
- Entity resolution given a fixed pair

**Properties:** cheap, deterministic-ish, fast, great for CI, easy to attribute. **Blind spot:** does not tell you how the components behave when composed (error propagation, tool choice).

### Regime B — Trajectory (process) evals
Score the sequence of steps for one run: which tools, which arguments, in what order, where it stopped.

**Properties:** catches loops, wrong-tool choices, termination bugs, wasted spend. **Blind spot:** a beautiful trajectory can still produce a wrong answer if the source data is wrong.

### Regime C — Outcome (end-to-end) evals
Score whether the user's task was accomplished, under production-like conditions, repeatedly.

**Properties:** the number your business actually cares about. **Blind spot:** tells you *that* it failed, never *why*.

**You need all three, and they must share the same case IDs and trace IDs** so you can walk from "task failed" down to "the entity link on step 6 picked the wrong John Smith."

```
Outcome      task success = 0
   ▲
   │  attribution
Trajectory   step 6: entity_lookup("John Smith") → node:person/8812 (wrong)
   ▲
   │  attribution
Component    evidence recall@10 = 0.0 for sub-query "John Smith 2019 acquisition"
   ▲
   │
Data         gold evidence exists; the linking model chose the most-connected node
```

That vertical walk — outcome → trajectory → component → data — is the entire discipline. Chapters 4 and 12 make it mechanical.

---

## 1.5 What "good" looks like, concretely

A team with a working eval system can answer these five questions in under a minute, for any release:

1. Did quality move, and on which slices?
2. Did cost or latency move?
3. Which failure bucket grew, and is it new?
4. Is the change within run-to-run noise?
5. What is the single highest-frequency failure we should fix next?

A team without one answers question 1 with an aggregate number and questions 2–5 with a meeting.

---

## 1.6 The trap to avoid on day one

Do not start by choosing a metric library. Start by **writing down the failures you have already seen** — support tickets, user complaints, your own bad demos. Ten real failures will teach you more about what to measure than any framework's metric list.

A useful opening exercise, one hour, with the team:

- List 10–20 real cases where the system gave a bad answer.
- For each, write one sentence: *what would have had to be true for this to go right?*
- Tally how many are retrieval, graph, reasoning, tooling, or data problems.

That tally is your first metric priority, and it will almost never match the metric list you would have picked from a blog post. Most teams discover their biggest bucket is "answered confidently from a stale document," which is an abstention/grounding problem, not a recall problem.

---

## Do this on your system

- [ ] Take your current eval suite and classify each metric as component, trajectory, or outcome. Note which regimes are missing. Most teams have only outcome.
- [ ] Pick three answers your system got *right* recently. Pull their traces (or reconstruct them). Ask: was the retrieval correct, or did the model get lucky?
- [ ] List your last 10 user complaints and bucket them. Keep the list; you will reuse it in Chapter 12.
- [ ] If you run a graph: for one multi-hop question, manually check whether the gold path actually exists in your graph. Do this five times. This is the single most discouraging and most useful hour you will spend.

## Common mistakes

- **Treating "the answer was right" as "the system worked."** Score the process, not just the artifact.
- **Reporting one blended quality score across question types.** Aggregates hide the slices that are actually broken.
- **Measuring only after the fact.** If you cannot replay a run step by step, you cannot evaluate an agent. Instrumentation comes first (Chapter 4).
- **Assuming the graph is an asset.** It is a model output with a cost and an error rate. It has to earn its place (Chapter 11).

## Key takeaways

1. Agentic RAG breaks the one-question/one-retrieval/one-answer assumption that most eval suites are built on.
2. A correct answer can hide a broken process; a broken process will eventually produce a wrong answer at scale.
3. Three regimes — component, trajectory, outcome — and you need all three, joined by shared IDs.
4. GraphRAG adds "graph wrong," "graph unused," and "graph not worth it" as first-class failure classes.
5. Start from your real failures, not from a metric catalog.

---

**Next:** [Chapter 2 — The eval stack](02-the-eval-stack.md), where we turn this mental model into a concrete metric map and an attribution procedure.
