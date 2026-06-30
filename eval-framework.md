# Evaluation Framework

## Why Evaluation Matters Here

In most AI applications, a wrong answer is an inconvenience. In health insurance, a wrong answer can mean a patient selects the wrong hospital room, misses a claim intimation deadline, or enters a hospital assuming coverage that doesn't exist.

The eval framework was designed around this asymmetry — errors in this domain have direct financial consequences, so the metrics reflect what actually matters to a policyholder.

---

## Test Setup

- **Version evaluated:** V0
- **Policies tested:** 6 synthetic policy documents (labelled O, A, B, C, D, E), generated using AI to cover a range of clause structures, insurer types, and complexity levels
- **Evaluation method:** Manual review against policy source documents using a structured Google Sheets rubric
- **Evaluation link:** [V0 Eval Sheet](https://docs.google.com/spreadsheets/d/1kK7Gq4IvonJQ_AiVhOXV7-NOdt5EjBIkqyTefpRmk58/)

Synthetic policies were used to enable controlled testing — real policies introduce confounds (OCR quality, document format variation) that are better addressed once the base model performance is established.

---

## Metrics

### Coverage
**What it measures:** The proportion of claim-critical fields present in the policy document that were successfully extracted into the output JSON.

**Why it matters:** A missed room rent cap or proportionate deduction clause is not a minor omission — it's the difference between a user knowing their risk or walking into a hospital uninformed. High coverage is the baseline requirement for the product to be useful.

**Target:** ≥ 90%

---

### Accuracy
**What it measures:** The proportion of extracted fields that are factually correct — matching the source document on value, condition, and scope.

**Why it matters:** A extracted room rent limit of ₹3,000 when the policy states ₹5,000 would cause a user to over-restrict their room selection unnecessarily, or worse, to calculate the wrong financial impact.

**Target:** ≥ 90%

---

### Hallucination Rate
**What it measures:** The proportion of extracted or generated fields that contain information not present in the source document.

**Why it matters:** Fabricated coverage, invented sub-limits, or made-up waiting periods are more dangerous than missing data. A user who believes a procedure is covered based on a hallucinated extraction may not purchase a top-up or rider that would have protected them.

**Target:** ≤ 10%

---

### Consistency
**What it measures:** Whether the same policy document, uploaded independently by different users, produces equivalent structured output.

**Why it matters:** PolicyWise AI is expected to deliver uniform intelligence across users on the same policy. If two members of the same family floater plan receive different sub-limit extractions, one of them has wrong information — and there is no way for either to know which.

**Target:** ≥ 90%

---

## V0 Results

### Aggregate

| Metric | V0 Score | Target |
|---|---|---|
| Coverage | 87% | ≥ 90% |
| Accuracy | 89% | ≥ 90% |
| Hallucination Rate | 12% | ≤ 10% |
| Consistency | 88% | ≥ 90% |

All four metrics are within 3 percentage points of target. No metric has cleared its threshold yet — FE–BE integration is gated on meeting all four.

---

### Per-Policy Breakdown

| Policy | Coverage | Accuracy | Hallucination Rate | Consistency |
|---|---|---|---|---|
| O | 84% | 90% | 9% | 92% |
| A | 92% | 84% | **31%** | 88% |
| B | 82% | 87% | 14% | 76% |
| C | 88% | 91% | 9% | 93% |
| D | 94% | 90% | 7% | 91% |
| E | 86% | 89% | 4% | 88% |

---

### Notable Findings

**Policy A — Hallucination outlier (31%)**
Policy A produced a significantly elevated hallucination rate compared to all other policies. The likely cause is non-standard document structure — Policy A used nested clause references and non-standard section headers that disrupted the extraction prompt's priority ordering. The model appeared to fill in expected fields from general insurance knowledge rather than the document, particularly for sub-limits and claim reduction factors.

This is being treated as a prompt robustness issue, not a data issue. The fix involves improving the extraction prompt's handling of ambiguous or non-standard document structures.

**Policy B — Consistency gap (76%)**
Policy B showed the lowest consistency score, driven by variance in how the proportionate deduction clause was extracted across runs — sometimes captured in full, sometimes truncated. This points to a chunking or attention issue on long clause text.

**Policies D and E — Best performers**
Policies D and E were the cleanest performers, with hallucination rates of 7% and 4% respectively. Both had standard IRDAI-aligned document structures with clear section headers, suggesting document structure is a significant variable in extraction quality.

---

## What Comes Next (V1 Eval)

- Expand to 15+ synthetic policies with deliberately varied document structures
- Introduce real policy documents (anonymised) once OCR quality is verified
- Add a 5th metric: **Extraction Completeness per Field Type** — to identify which specific fields (e.g. sub-limits vs waiting periods) underperform, rather than aggregating across all fields
- Automate scoring where possible to reduce manual review overhead
