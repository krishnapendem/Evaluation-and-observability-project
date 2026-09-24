# Reflection Brief — Evaluation and Observability Capstone

**Name:** Sahil Aggarwal
**Date:** 2026-09-24

> Ground every answer in your own run. When a question asks for a number, file name, or line, paste
> it from your artifacts — a reviewer should be able to find it. Answers that are correct in the
> abstract but cite nothing do not meet the bar. Keep it short and specific.

---

## 0. Environment

| Field | Value |
|---|---|
| OS & version | macOS Darwin 25.6.0 (arm64) |
| Python version | Python 3.11.5 |
| Date run | 2026-09-24 |
| Ran any system live? (which) | No. Live Anthropic runs were blocked by missing API auth; the evidence pack uses the repo’s offline/fixture workflow and generated validation artifacts. |

---

## 1. Validated, routed pipeline

| Evidence | Value |
|---|---|
| Passing test count | 45 passed, 3 skipped |
| Routing output file | `Project-Evaluation and Observability Project/capstone-submission/01-policy-pipeline/routing_decisions.json` |
| auto_approve / human_review / spot_check counts | 1 / 2 / 1 |

**1a. Retry boundary.** From your perturbation run (a required field removed), paste the escalation
record. How many API calls did the system make, and why is retrying a futile case worse than
escalating it?

> `{"kind":"escalation","policy_id":"POL-EMPTY","field":"coverage_limit","reason":"missing required field","retries_used":1,"status":"retry_futile_escalation"}`
>
> The retry boundary is a futile case because a required field is structurally absent; retrying just replays the same invalid input and burns API budget without improving truth. Escalation preserves the signal and moves the item to a human review queue instead of looping on a guaranteed failure.

**1b. Reading the router.** Pick one `human_review` record from your routing output. Which of the
three signals (confidence, reviewer, integration) sent it to a human? If you had trusted the model's
confidence alone, what would have happened?

> The `POL-101` record in `routing_decisions.json` is `human_review` because `premium_amount` confidence is `0.65`, below the `0.90` threshold. The reviewer and integration checks were clean, but the confidence signal alone would have auto-approved it because the other fields were all `0.99`.

**1c. Where the aggregate lies.** Run the calibration snippet. Quote the one cell whose accuracy lags
its confidence, plus the overall figure. What does slicing by `policy_type × field` catch that a
single number hides?

> `umbrella / exclusions`: `conf=0.93 acc=0.00 brier=0.865` and `OVERALL brier=0.291` from `capstone-submission/01-policy-pipeline/calibration-report.txt`.
>
> A single aggregate can look healthy while one slice is consistently wrong. The slice shows the `umbrella/exclusions` risk that is hidden inside the broader average.

---

## 2. Schema-enforced two-pass extraction

| Evidence | Value |
|---|---|
| Passing test count | 25 passed |
| Document run | `fixtures/documents/appraisal_informal_sqft.txt` |
| Classified type | `single_family` property; normalized `gross_living_area_sqft` to `2400` |

**2a. Two guarantees.** Paste your discrepancy-run output. Tool use already forces valid JSON, yet the
validator still catches a bad sum. Why are these two different guarantees? Name one error each cannot
catch.

> From `capstone-submission/02-mortgage-extraction/discrepancy-run.txt`:
> `"field": "total_monthly_income", "calculated": 9642.17, "stated": 10892.17, "delta": -1250.0` and `"consistent": false`.
>
> Tool use guarantees syntactic validity (`JSON` is well-formed), while validation guarantees semantic plausibility (`calculated_total` vs `stated_total` matches the contract). Tool use cannot catch a logically wrong numeric sum in a valid payload; validation cannot catch a malformed schema or a missing required field that the schema itself says must be present.

**2b. Refusing to fabricate.** Run on a document missing a field. Paste that field's output. Why null
instead of an invented value? Point to the schema choice that allows it.

> From `capstone-submission/02-mortgage-extraction/missing-field-run.txt`:
> `"stated_monthly_total": null`
>
> The schema explicitly allows nullable fields for missing or unstated values, so the extractor returns `null` rather than inventing a number. This is the schema-level contract that preserves honesty under incomplete source data.

**2c. Normalization.** Quote one field where the source text and extracted value differ in format
("about 2,400 sq ft" → `2400`). Why normalize at extraction time rather than downstream?

> `"gross_living_area_sqft": 2400` in `extract-run.txt` from `appraisal_informal_sqft.txt`.
>
> Normalization at extraction time creates one canonical unit for downstream validation and calculations, so the validator compares numbers in the same unit instead of re-parsing text at each later stage.

---

## 3. Multi-source synthesis

| Evidence | Value |
|---|---|
| Passing test count | 34 passed in 60.14s |
| Briefing file | `Project-Evaluation and Observability Project/capstone-submission/03-supply-chain/briefing.txt` |
| Section the conflict landed in | `Contested` |

**3a. Annotate, don't arbitrate.** Quote one conflicting-metric pair from your briefing — both values,
sources, dates. Give one way a reader is better served by the preserved conflict than by a single
reconciled number.

> In `briefing.txt`: `95.0 percent — supplier_audit (as of 2026-04-10)` vs `78.0 percent — logistics (as of 2026-04-05)` for `on_time_delivery_rate`.
>
> Preserving the conflict tells the reader that the two sources disagree and that the gap is date-sensitive; a single reconciled number would hide which source is newer and which operational issue is driving the difference.

**3b. Source goes dark.** Run with `--simulate-timeout`. Paste the part of the briefing showing the
failed source. How is "unreachable" handled differently from "nothing to report," and why does the run
still finish?

> From `capstone-submission/03-supply-chain/timeout-run.txt`:
> `Sources unavailable: logistics unavailable (timeout)` and `late_shipment_count _[missing source: timeout reading logistics]`.
>
> "Unreachable" is tracked explicitly as an unavailable source and annotated in the `Incomplete` section; it is not silently treated as a clean zero or as “no data.” The coordinator still finishes because it continues with all remaining sources and records the missing source as a gap instead of aborting the run.

**3c. Dates as a guardrail.** Quote two claims about the same supplier with different dates. How does
requiring a date stop a time difference from reading as a contradiction?

> `supplier_audit (as of 2026-04-10)` vs `logistics (as of 2026-04-05)` for `on_time_delivery_rate`.
>
> The date field makes the conflict temporal: these are not two simultaneous claims but two observations from different points in time. The system therefore records a dated conflict rather than a false contradiction.

---

## 4. Synthesis

**4a. One principle.** Name the single moment in your runs (system + artifact) where *evaluate the
output, don't trust the model's word* most clearly caught something a trusting design would have
shipped.

> The clearest moment was the mortgage validator: `capstone-submission/02-mortgage-extraction/discrepancy-run.txt` reported `"consistent": false` with `total_monthly_income` at `9642.17` vs `10892.17`. A trusting design would have shipped the valid-looking JSON without noticing an arithmetic mismatch.

**4b. Confidence ≠ correctness.** Pick the system where this mattered most, and explain why using
something you observed.

> The policy routing system mattered most. In `routing_decisions.json`, `POL-101` was a `human_review` because `premium_amount` was `0.65` even though the rest of the fields were `0.99`. Confidence alone would have auto-approved the record and hidden a materially risky field.

**4c. Apply it.** Describe a real workflow where an LLM pulls structured results from messy input.
Which pattern — validated retry with escalation, independent review with deterministic routing, or
provenance-preserving conflict annotation — would you reach for first, and what would you instrument
to know when it broke?

> For an insurance intake workflow, I would reach for validated retry with escalation first. I would instrument retry count, escalation count, missing-field rate, validation-failure rate, and the distribution of confidence by field so I can tell when a field is failing structurally or semantically before it reaches a downstream approval step.
