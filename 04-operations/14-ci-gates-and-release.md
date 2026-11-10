# Chapter 14 — CI gates and release process

> **Chapter 14** · Part IV: Operating it
> **Time:** 20 minutes
> **You will leave with:** a four-tier test pyramid, gate thresholds you can defend, and a release process that blocks the right things.

---

## 14.1 The eval pyramid

Match the eval to the frequency of the change. Running the full suite on every PR is how teams turn a 40-minute wait into a reason to disable CI.

| Tier | Trigger | Cases | Runtime | Runs | Blocks? |
|---|---|---|---|---|---|
| **T0 — unit/contract** | every commit | ~100 assertions on pipeline invariants | < 30s | No LLM calls | Yes |
| **T1 — smoke eval** | every PR | 40–60 cases (slice coverage + prioritised known regressions) | 3–5 min | Deterministic + 1 judge | Yes |
| **T2 — full regression** | nightly + pre-release | 150–300 cases, pass^1; pass^3 on 30 core | 30–90 min | Full | Release |
| **T3 — canary** | deploy | live traffic slice | 1–24h | Production | Rollback |

### T0 is where most value hides

Deterministic pipeline assertions catch an enormous share of real bugs in seconds:

```python
# Contract tests — no model calls, no cost
def test_every_retrieval_span_has_chunk_ids(trace): ...
def test_answer_requiring_citations_has_valid_citations(trace): ...
def test_no_retrieved_chunk_outside_user_acl(trace): ...
def test_graph_nodes_have_provenance(): ...            # on a 50-node fixture
def test_entity_resolution_no_contradictory_attributes(): ...
def test_incremental_rebuild_removes_stale_edges(): ...  # fixture corpus
def test_no_duplicate_chunks_in_index(): ...
def test_prompt_template_renders_without_missing_vars(): ...
def test_output_schema_valid(answer): ...
```

Teams consistently under-invest here and over-pay for judges to rediscover broken JSON.

---

## 14.2 Gate design

### Absolute floors and relative regressions

A gate needs both. One catches "we broke it"; the other catches "we let it rot."

```yaml
absolute:
  task_success:        { min: 0.70 }
  claim_support:       { min: 0.90 }
  acl_violations:      { max: 0 }        # hard zero
  injection_successes: { max: 0 }        # hard zero
  p95_latency_ms:      { max: 6000 }
  cost_per_success:    { max: 0.06 }

relative:                                 # vs. main branch, same eval batch
  task_success:        { max_drop: 0.03 }
  slice_success:       { max_drop: 0.08, min_slice_n: 50 } # below 50: report, don't gate
  pass3:               { max_drop: 0.05 }
  evidence_recall:     { max_drop: 0.04 }
```

Design notes:

- **Per-slice guard.** Without it, an 8-point loss on multi-hop is hidden by a 3-point gain on single-hop. Aggregates are how slices die.
- **Hard zeros on ACL and injection.** These are not trade-offs.
- **Latency and cost are gates**, not dashboards. If they only appear on a chart, they regress.
- **Choose practical thresholds at supported precision.** A 3-point drop is `0.03`, not `3`; avoid false precision such as a floor of `0.7314`. A sample-size floor alone does not establish power.

### Statistical significance

A 3-point difference on 150 cases may be noise or a repeatable change; the paired outcomes determine its uncertainty. Overlap of two separate confidence intervals is not a test of their difference.

- Compare candidate and baseline on the same cases in the same batch. Bootstrap **paired case-level differences** and report a CI for Δ. If runs are repeated, keep all runs for a case together when resampling; do not count them as independent questions. For binary single-run outcomes, an exact paired test such as McNemar's test is another option.
- Declare the decision rule before running. A practical default blocks a relative regression when the observed drop exceeds the tolerance and the paired interval excludes zero. To claim the drop exceeds the tolerance with 95% confidence, require the entire interval to lie below `−tolerance`. For many slice gates, prespecify the comparisons and control multiplicity.
- Apply absolute floors and cost/latency ceilings separately; an absolute breach does not require a significant change from baseline. Report uncertainty and use a predefined escalation or expansion policy for borderline results.
- For stochastic systems, passing a significance test does not establish non-inferiority. An interval spanning both acceptable and harmful changes is **inconclusive**: expand the sample or use an explicit review policy. No fixed `min_slice_n` guarantees enough power.

Use T0 for deterministic assertions, T1 for fast regression checks, and T2 for paired estimates with sufficient cases and runs.

### Flake handling

Predeclare which cases are strict invariants and which use stochastic rate gates.

```
Hard-zero violation or deterministic contract failure:
  block immediately; reruns do not erase the failure.

Known-regression / strict core case:
  schedule 3 runs for every case in advance.
  all 3 pass → pass^3 success for that case.
  any failure → block pending triage; mixed outcomes also enter the flip set.

Other stochastic cases:
  use the scheduled run budget and the aggregate/slice gate policy.
  diagnostic reruns may aid triage; retain the original observations.
```

Running extra attempts only after failure is a diagnostic retry policy, not pass^3 measurement. Never replace a failed observation with a passing retry. Mixed outcomes are evidence of unreliability, and cannot automatically waive a strict regression or hard-zero gate.

---

## 14.3 What actually blocks a release

Write this down and post it. Ambiguity here is how gates get overridden under deadline pressure.

**Blocks:**
1. Any hard-zero violation (ACL, injection, safety).
2. T1 smoke failure on a known-regression case.
3. T2 aggregate drop beyond threshold with CI excluding zero.
4. T2 per-slice regression under the same declared interval rule and sample-size policy.
5. Cost or p95 latency beyond the absolute ceiling.
6. A judge change without re-calibration.

**Does not block, but requires a written note in the PR:**
- A slice improving while another regresses within thresholds.
- A new failure bucket appearing at low frequency.
- `unclassified` rate rising.
- Any pass^3 drop within threshold.

**Explicitly allowed:**
- An explicitly reviewed trade-off between non-critical buckets, documented with an owner and follow-up. A gate override must be recorded by the designated release owner; hard-zero violations are not waived by this policy.

---

## 14.4 The release checklist

```
[ ] T0 green
[ ] T1 green (40–60 cases, known regressions included)
[ ] T2 run in the last 24h, on the exact artifact being shipped
[ ] Judge version unchanged, or re-calibrated
[ ] Eval set version unchanged, or changelog reviewed
[ ] Cost/success and p95 within ceilings
[ ] Canary plan: slice %, primary metric, guardrails, rollback trigger
[ ] Known regressions documented with owners
```

Two rules that prevent most eval-process failures:

- **Eval the artifact you ship.** Running T2 on `main` and shipping a branch is a classic. Pin the commit in the report.
- **One change class per release.** If you change the prompt, the retriever, and the model together, the eval tells you the sum and nothing else.

---

## 14.5 The eval-driven development loop

The productive daily loop, once the infrastructure exists:

```
1. Pick the top bucket from the weekly review.
2. Write/confirm the failing case(s). They must be red.
3. Change one thing.
4. Run T1 (fast). Iterate.
5. Run T2 when T1 is green and the change looks real.
6. Check the paired slice table: did you fix X without breaking Y?
7. Ship behind canary. Watch the proxy metric for 24h.
8. If it worked, the case joins the permanent regression set.
```

Step 2 is the discipline. Fixing before writing the case produces tests that assert current behaviour, which is the opposite of what you want.

---

## 14.6 Dashboards that get used

Three views, nothing more:

1. **Release gate report** (per release, one screen): gates pass/fail, slice deltas, cost/latency, failure mix.
2. **Trend** (weekly): north star + three guardrails over 12 weeks, with annotations for releases. Annotations are what make trends interpretable.
3. **Failure queue** (live): flip set, unclassified traces, open bucket tickets.

Anything else is a museum. If a chart has not changed a decision in a month, delete it.

---

## Do this on your system

- [ ] Write the T0 contract tests. Aim for 30+ assertions in under 30 seconds.
- [ ] Assemble the 40-case T1 set from your known regressions, one per slice.
- [ ] Write `gates.yaml` with absolute floors, relative drops, and hard zeros.
- [ ] Implement the 3-run flake policy.
- [ ] Post the "what blocks a release" list where the team can see it.
- [ ] Add the release checklist to your PR template.
- [ ] Delete every dashboard nobody has acted on.

## Common mistakes

- **Running the full suite on every PR.** It gets disabled within a month.
- **Gating on aggregates only.** Slices regress invisibly.
- **Treating stochastic rates as deterministic after one run.** Plan repeated sampling; retain immediate blocks for hard-zero and contract failures.
- **No hard zeros.** ACL and injection must be non-negotiable, not "within tolerance."
- **Evaluating a different commit than you ship.**
- **Changing the judge and the system together.** You cannot attribute the result.
- **Gates that only warn.** A warning is a decision to ship.

## Key takeaways

1. Four tiers: contract tests (seconds), smoke (minutes), full regression (nightly), canary (hours).
2. Gate on absolute floors, relative drops, and per-slice guards; hard-zero ACL and injection.
3. Schedule repeated runs in advance; any failure blocks strict cases, and mixed outcomes enter the flip set. Diagnostic retries never erase failures.
4. Publish exactly what blocks a release, in writing.
5. Eval the commit you ship, and change one class of thing per release.

---

**Next:** [Chapter 15 — Tooling: build vs buy](15-tooling-build-vs-buy.md). A short, opinionated map.
