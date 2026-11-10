# Chapter 6 — Generation evals

> **Chapter 6** · Part II: Component evals
> **Time:** 25 minutes
> **You will leave with:** a claim-level scoring method, the faithfulness/correctness 2×2, and an abstention metric that is not gameable.

---

## 6.1 Why whole-answer scores fail

"Faithfulness: 0.87" on a five-sentence answer means nothing. Which sentence? The one that mattered, or the boilerplate?

Two structural problems with whole-answer scoring:

1. **Averaging hides the fatal clause.** One unsupported number in a compliance answer is the whole problem. A mean over sentences launders it.
2. **It conflates support and truth.** An answer can be perfectly supported by retrieved context and still be wrong, if the context was wrong or stale.

Fix both by scoring at the **claim** level and by reporting support and correctness separately.

---

## 6.2 Claim decomposition

A *claim* is an atomic, checkable assertion. Split the answer first, then score each claim.

```
Answer: "Northwind Logistics appears in the 2023 audit findings for late
         delivery, and we renewed their contract in March 2024."

Claims:
  c1: Northwind Logistics appears in the 2023 audit findings.
  c2: The audit finding concerns late delivery.
  c3: The Northwind contract was renewed in March 2024.
```

Decomposition rules that keep judges stable:

- One predicate per claim. Split conjunctions.
- Resolve pronouns to entities.
- Keep numbers, dates, and units exactly. Preserve approximation: "about 4.2M" and "exactly 4.2M" have different precision; "$4.2M" and "4.2M units" are not.
- Do not split into sub-clauses so fine that entailment becomes ambiguous. Two or three claims per sentence is the sweet spot.
- Skip hedges and meta-commentary ("Based on the documents…"). They are not claims.

Decomposition can be done by an LLM with a fixed prompt (Appendix C) or, for structured outputs, by schema. Cache it: LLM decomposition can vary even at low temperature. Cache by answer, decomposition prompt/version, model, and settings so repeat scoring uses the same claims.

---

## 6.3 Claim support rate (faithfulness)

> Of the claims in the answer, what fraction is entailed by the retrieved context?

```
claim_support_rate = supported_claims / total_claims
```

A bare abstention has no factual claims: report support as `None`, not 0 or 1. An explanation such as "the policy was withdrawn in 2024" does contain a claim; score it normally and score the abstention decision separately. Mixing empty answers into the support average is a quiet way to make the metric look better or worse than it is.

Report three variants, because they answer different questions:

| Variant | Definition | Use |
|---|---|---|
| **Context support** | Claim entailed by the context actually delivered to synthesis, possibly combining chunks | Classic faithfulness. Catches hallucination. |
| **Citation support** | Claim entailed by its own cited chunk(s); uncited claims count as unsupported | Catches miscitation, which context support hides |
| **Strict support** | Every required claim fully citation-supported, with no unsupported qualifiers; binary per answer | Compliance / regulated answers |

The gap between context support and citation support is a real and under-reported failure: the answer is grounded, but the citation points at the wrong paragraph. Users click it, see nothing relevant, and stop trusting citations. If you show citations in your UI, **citation support is the metric that matters**, not context support.

Use three verdicts, not a 1–10 scale: `supported` / `partially_supported` / `unsupported`. A 7-point scale invites judge drift and inter-rater disagreement. If you need a continuous number, compute `(supported + 0.5·partial) / total` explicitly and label it as a derived score.

---

## 6.4 The 2×2 that explains most arguments

Teams argue about whether the model or the retrieval is at fault. Settle it with a two-axis report.

|  | **All factual claims supported** | **At least one claim unsupported** |
|---|---|---|
| **Answer fully correct and complete** | Working as intended | Investigate parametric knowledge, incomplete evidence labels, or support-grader error |
| **Answer incorrect or incomplete** | Investigate stale/incorrect evidence, omitted facts, or task mismatch | Investigate retrieval gaps, generation errors, and support-grader error |

Report the counts in all four cells, per slice. Then:

These cells guide investigation; they do not prove a root cause. A faithful answer can omit a required fact even when every source is correct (Appendix D). An unsupported answer can follow a retrieval miss as well as a generation error. Inspect the evidence and replay with fixed gold context before assigning ownership. Use answer-level support status for this table; claim-level rates and answer-level correctness have different units.

---

## 6.5 Correctness

Correctness is measured against `gold_answer`, and it needs a definition per task type:

| Task type | Correctness definition | Method |
|---|---|---|
| Extractive / short answer | Exact match after normalisation (aliases, units, casing) | Deterministic |
| Set answer ("which suppliers…") | Set F1 over required entities | Deterministic |
| Numeric | Within tolerance and correct units | Deterministic |
| Free-form explanation | All required facts present, no contradicting facts | Judge + required-facts rubric |
| Refusal expected | Correctly declined without asserting a guessed answer | Check structured behaviour flag plus semantic review of the text |

**Do not use ROUGE or BLEU** for answer correctness on free text. They reward surface overlap, punish correct paraphrases, and are trivially gamed by repeating the question. They remain acceptable for *summarisation against a reference summary* when you report them beside a semantic metric — and even then, be aware that reference-based metrics favour detail fidelity over coverage, which systematically disadvantages corpus-level answers.

For free-form answers, use a **required-facts rubric** plus forbidden facts and reference evidence. `found_required / total_required` measures completeness. Fully correct requires every required fact, no forbidden or contradicted assertion, and no incorrect extra claim. A finite forbidden list alone cannot catch every false extra detail.

---

## 6.6 Completeness

Correctness asks "is what's here true?" Completeness asks "is everything here?"

Multi-hop questions fail by *partial answer* far more often than by wrong answer: two of three suppliers named, one of two conditions stated. This is invisible to a correctness check that rewards the presence of true content.

```
completeness = required_facts_present / required_facts_total
```

Track it separately and report the fraction of cases that are both correct and complete (`correct ∧ complete`), not `mean(correctness) × mean(completeness)`. That joint rate is usually 10–20 points below either component, and it is the one users experience.

---

## 6.7 Abstention and calibration

The highest-trust behaviour in a RAG product is a well-calibrated "I don't know." It is also the easiest to get wrong in both directions.

Score a 3×2 confusion matrix over answerable and unanswerable cases:

|  | Answered | Abstained |
|---|---|---|
| **Answerable** | Check correctness separately | **over-rejection** (false abstain) |
| **Unanswerable** | **false answer** (hallucinated) | correct rejection |
| **Ambiguous** | Evaluate clarification separately | A bare refusal may also fail |

Report:

- `false_answer_rate` on unanswerable cases — the trust-killer.
- `over_rejection_rate` on answerable cases — the usefulness-killer.
- **Calibration**: define confidence as a probability of the scored success event, then compare confidence and empirical accuracy in bins. Raw retrieval similarity is not such a probability; learn and validate any mapping on held-out data. Report bin sizes and uncertainty, including for abstentions if scoring all decisions.

**Why a single "abstention accuracy" is gameable:** a system that abstains on everything scores 100% on the unanswerable slice. Always report the pair. The acceptable operating point is chosen by product, not by the model: set the cost of false answers and unnecessary refusals for the specific workflow, with domain owners.

---

## 6.8 Consistency and robustness

Cheap checks that catch real regressions:

- **Paraphrase consistency** — ask the same question three ways; the required facts should be stable. Fluctuation > 30% signals brittle retrieval or a temperature/prompt issue.
- **Order robustness** — shuffle the retrieved chunks. A large change in the answer means the model is position-sensitive and your ranking is doing more work than you think.
- **Distractor resistance** — inject one plausible but wrong chunk. Correct behaviour: ignore it, or explicitly note the conflict. This is the single best test for over-trusting retrieval.
- **Contradiction handling** — provide two chunks that disagree (v1 vs v2 policy). Correct behaviour: prefer the newer/authoritative source and say so, or flag the conflict. Most systems silently pick one.

---

## 6.9 Targets

| Metric | Unacceptable | Acceptable | Good |
|---|---|---|---|
| Claim support (context) | < 0.85 | 0.90–0.95 | > 0.97 |
| Citation support | < 0.70 | 0.80–0.90 | > 0.95 |
| Correctness (fully correct) | < 0.55 | 0.65–0.80 | > 0.85 |
| Completeness (multi-hop) | < 0.55 | 0.65–0.80 | > 0.85 |
| False-answer rate (unanswerable) | > 0.25 | 0.08–0.15 | < 0.05 |
| Over-rejection rate (answerable) | > 0.20 | 0.05–0.12 | < 0.05 |
| Distractor flip rate | > 0.30 | 0.10–0.20 | < 0.08 |

Note that claim support above 0.90 is normal and easy; the hard part is citation support and the abstention pair. Systems that report 0.97 faithfulness while misciting half their claims are common.

---

## Do this on your system

- [ ] Add claim decomposition + support scoring to your nightly eval (Appendix C has a prompt).
- [ ] Compute the 2×2 table for your current failures. Identify which cell dominates.
- [ ] Switch free-text correctness to a required-facts / forbidden-facts rubric.
- [ ] Measure citation support separately from context support. If you show citations, this is your real number.
- [ ] Run the distractor-injection test on 30 cases.
- [ ] Report false-answer and over-rejection rates as a pair, never alone.

## Common mistakes

- **Whole-answer faithfulness scores.** They hide the one clause that matters.
- **Context support standing in for citation support.** Grounded but miscited is still a broken product.
- **1–10 judge scales.** Use three verdicts; derive a continuous score if you must.
- **ROUGE as correctness.** Rewards paraphrase failure and repetition.
- **Abstention scored as one number.** Trivially gamed by refusing everything.
- **Assigning causes from the 2×2 alone.** Supported but incomplete answers can reflect synthesis omissions as well as bad or stale data.

## Key takeaways

1. Decompose into claims; score each. Whole-answer averages launder fatal clauses.
2. Report context support, citation support, and strict support separately.
3. The support × correctness 2×2 suggests hypotheses; confirm causes with evidence inspection and fixed-context replays.
4. Completeness is a separate axis; multi-hop answers usually fail partially, not wrongly.
5. Abstention is a pair of rates plus a calibration curve. Never a single number.

---

**Next:** [Chapter 7 — Graph construction evals](07-graph-construction-evals.md). The part of GraphRAG nobody measures and everybody blames.
