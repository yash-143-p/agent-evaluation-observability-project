# Reflection Brief — Evaluation and Observability Capstone

**Name:** Yashwanth P
**Date:** 2026-09-28

> Ground every answer in your own run. When a question asks for a number, file name, or line, paste
> it from your artifacts — a reviewer should be able to find it. Answers that are correct in the
> abstract but cite nothing do not meet the bar. Keep it short and specific.

---

## 0. Environment

| Field | Value |
|---|---|
| OS & version | Linux x86_64 |
| Python version | 3.13.0 |
| Date run | 2026-09-28 |
| Ran any system live? (which) | Yes — Policy pipeline perturbation run completed successfully. |

---

## 1. Validated, routed pipeline

| Evidence | Value |
|---|---|
| Passing test count | 45 passed, 3 skipped |
| Routing output file | `01-policy-pipeline/perturbed-routing-decisions.json` |
| auto_approve / human_review / spot_check counts | 0 / 1 / 0 |

The Policy test suite completed with 45 passed and 3 skipped. The routing tests themselves passed, including tests for auto-approve, human review, integration failure, reviewer disagreement, spot checking, calibration slicing, and JSON routing output. The captured test suite reports `45 passed, 3 skipped`. 

### 1a. Retry boundary

I deliberately removed the `Named Insured` value from a copy of `POL-2025-001.txt` and ran the end-to-end pipeline against the perturbed document.

The final live run completed successfully. The captured run shows two successful API calls and the router recorded a reviewer disagreement on `coverage_limit`. The routing summary reported:

- `decisions_written`: `1`
- `auto_approve`: `0`
- `human_review`: `1`
- `spot_check`: `0`
- `escalations`: `0`

The final routing record has `decision: human_review` and `reason: reviewer_disagreement=['coverage_limit']`. The record also has `fields_below_threshold: []`, so I do not claim that the missing `Named Insured` field itself was the direct routing trigger.

### 1b. Reading the router

The actual routing artifact is `01-policy-pipeline/perturbed-routing-decisions.json`. It contains one record for `POL-2025-001` with `decision: human_review` and `reviewer_disagreements: ["coverage_limit"]`. The confidence summary includes `coverage_limit: 1.0`, while the explicit routing reason is the independent reviewer disagreement.

This provides the required evidence that an independent signal can route a case to human review rather than allowing the extraction to be silently accepted.

### 1c. Where the aggregate lies

The calibration report produced the following sliced results:

`umbrella  exclusions  n=2 conf=0.93 acc=0.00 brier=0.865`

The overall result was:

`OVERALL brier=0.291`

The sliced view matters because an aggregate can hide a poorly calibrated policy-type/field combination. In this run, the umbrella/exclusions slice had high confidence but zero observed accuracy, while the overall aggregate alone would not reveal that specific weakness.

---

## 2. Schema-enforced two-pass extraction

| Evidence | Value |
|---|---|
| Passing test count | 25 passed |
| Document run | `02-mortgage-extraction/extract-run.txt` |
| Classified type | Not explicitly printed in the captured extraction artifact |

### 2a. Two guarantees

The discrepancy run produced:

`consistent: false`

`calculated: 9642.17`

`stated: 10892.17`

`delta: -1250.0`

These are different guarantees. Structured/tool-based extraction ensures that the returned data conforms to the expected schema and data structure. Deterministic validation checks whether the extracted values make sense together mathematically.

Schema enforcement could catch an invalid structure or wrong field type, but it would not necessarily catch a correctly formatted numeric value whose relationship to other values is wrong. Conversely, a mathematical validator cannot by itself guarantee that every extracted field has the correct schema or structure.

### 2b. Refusing to fabricate

The missing-field run produced:

`"bonus_monthly": null`

It also showed `bonus_ytd`, `commission_monthly`, `overtime_monthly`, `other_monthly`, and `stated_monthly_total` as `null` where those values were not available.

Using `null` preserves the distinction between information that was not present and information that was actually extracted. This prevents the system from inventing a value that was not supported by the source document.

### 2c. Normalization

The normal extraction artifact contains:

`"gross_living_area_sqft": 2400`

The extracted value is represented as a numeric field rather than as a formatted natural-language measurement. Normalization at extraction time gives downstream validation and processing a consistent machine-readable representation.

The captured artifact does not contain the original phrase used in the source document, so I do not claim an exact source-text quotation that is not present in the evidence file.

---

## 3. Multi-source synthesis

| Evidence | Value |
|---|---|
| Passing test count | 34 passed |
| Briefing file | `03-supply-chain/briefing.md` |
| Section the conflict landed in | `Contested` |

### 3a. Annotate, don't arbitrate

The normal Meridian briefing reports:

- `95.0 percent` — supplier_audit, as of `2026-04-10`
- `78.0 percent` — logistics, as of `2026-04-05`

for `on_time_delivery_rate`.

The briefing places this metric in the `Contested` section and marks it for escalation because it is a high-impact metric with conflicting sources.

Preserving both values lets a reader see the disagreement and investigate source dates and provenance. Replacing them with one reconciled number would hide the fact that the sources disagree and could make the reader believe there was stronger agreement than the evidence actually showed.

### 3b. Source goes dark

With the simulated timeout, the run reported:

`Sources unavailable: logistics unavailable (timeout)`

and marked the affected information as:

`late_shipment_count [missing source: timeout reading logistics]`

The system therefore distinguishes an unavailable source from a source that simply did not report a particular metric. In the normal run, `production_capacity_utilization` was marked incomplete because no source reported that metric. During the timeout run, the logistics-dependent information was marked incomplete because the source itself was unavailable.

The run still finishes because the coordinator preserves the available evidence and explicitly annotates the unavailable portion instead of treating the entire investigation as a failure.

### 3c. Dates as a guardrail

For `defect_rate_ppm`, the briefing records:

- `180.0 ppm` — supplier_audit, as of `2026-04-10`
- `190.0 ppm` — internal_quality, as of `2026-04-08`

The dates show that these observations were made at different times. Requiring dates prevents a difference between observations at different points in time from automatically being interpreted as a contradiction.

---

## 4. Synthesis

### 4a. One principle

The clearest example was the mortgage extraction discrepancy run in `02-mortgage-extraction/discrepancy-run.txt`.

The extracted income values were structurally valid, but deterministic validation found:

`consistent: false`

with a calculated total of `9642.17` versus a stated total of `10892.17`, producing a delta of `-1250.0`.

A system that trusted the extracted value without independently validating the relationship between fields could have shipped the inconsistent total.

### 4b. Confidence ≠ correctness

The Policy calibration evidence demonstrates this clearly. The `umbrella / exclusions` slice had:

`conf=0.93`

but:

`acc=0.00`

with:

`brier=0.865`

This shows why confidence from a model should not be treated as equivalent to correctness. Evaluation and sliced calibration expose situations where a model can be highly confident while still being wrong on a particular policy type and field.

### 4c. Apply it

A real workflow would be an insurance or financial-document processing system where an LLM extracts structured information from policies, applications, invoices, or other semi-structured documents.

For a workflow where source evidence can conflict, I would use provenance-preserving conflict annotation so that each extracted value retains its source and date and disagreements remain visible.

I would instrument:
- extraction validation failures,
- missing fields,
- source identifiers,
- source dates,
- conflicting values,
- incomplete sources,
- escalation counts,
- and final validation outcomes.

These signals would make it possible to detect when the system is producing structured output that is incomplete, inconsistent, or supported by conflicting evidence.
