# Confidence Scoring
**Ledger Lens Reliability Patterns — Confidence Scoring for LLM Extraction**

---

## Status
**Pattern maturity:** Field-tested design  
**Reference implementation:** Pending (Ledger Lens open-source release)  
**Last updated:** v0.1

> This document describes a confidence scoring approach derived from operating LLM-assisted document extraction systems on noisy real-world data (OCR text, semi-structured filings, heterogeneous schemas).  
> The Ledger Lens project will provide a clean open reference implementation of these patterns.

---

## 1. Why Confidence Scoring Exists

LLM extraction systems do not fail loudly.

They fail **plausibly**.

Traditional software:
Parse error → exception → retry → alert

LLM systems:
Incorrect company number → looks valid → stored silently → decision corruption

This creates a new failure class:

> **Semantic failure with syntactic validity**

Confidence scoring exists to convert silent semantic failure into measurable risk.

---

## 2. What Confidence Is (and Is Not)

### Not confidence
- Model probability
- Token likelihood
- Logprobs
- Temperature heuristics

These measure **model certainty**, not **truthfulness**.

### Real confidence
Confidence estimates **trustworthiness of extracted data** given external evidence.

> **Confidence = Agreement between independent signals about the same fact**

---

## 3. Core Principle — Multi-Signal Agreement

Every extracted field must be supported by multiple independent signals.

| Signal Type | Example |
|-----------|------|
| Structural | Regex / format validity |
| Source grounding | Appears in authoritative dataset |
| Contextual | Surrounding text supports value |
| Cross-document | Appears in multiple filings |
| Model consensus | Multiple prompts/models agree |

Confidence increases when signals agree and collapses when they diverge.

---

## 4. Field-Level Scoring Model

We do not score documents.  
We score **each extracted field individually**.

Example output:

```json
{
  "company_number": {
    "value": "12345678",
    "confidence": 0.96,
    "signals": ["regex_valid", "authoritative_match", "context_match"]
  },
  "insolvency_date": {
    "value": "03/07/2022",
    "confidence": 0.62,
    "signals": ["context_match"],
    "warnings": ["ambiguous_format"]
  }
}
```

This allows partial trust instead of binary acceptance.

---

## 5. Signal Categories

### 5.1 Structural Validation

Checks whether the value could exist.

Examples:
- Company number length
- Date validity
- Currency format

**Purpose**: reject impossible values cheaply.

### 5.2 Authoritative Grounding

Compare extracted value to trusted source.

Examples:
- Public registries
- Internal master data
- Known identifiers

**Purpose**: verify factual correctness
This is the strongest signal.

### 5.3 Contextual Support

Verify surrounding language supports the extracted value.

Example:

> “Company number: 12345678 entered administration”

vs

> “Reference example company 12345678”

Purpose: detect misplaced extraction.

### 5.4 Cross-Source Agreement

Check if multiple documents agree.

Consistent repetition across independent documents increases confidence.

**Purpose**: detect OCR noise and parsing drift.

### 5.5 Model Consensus (Optional)

Ask the model twice using different prompts or reasoning styles.

Agreement increases confidence.
Disagreement triggers review.

**Purpose**: detect fragile reasoning.

---

## 6. Confidence Aggregation

Confidence is not averaged.
It is bounded by the weakest critical signal.

Example rules:
- If authoritative check fails → max confidence = 0.3
- If format invalid → confidence = 0
- If only contextual support → medium confidence
- If authoritative + structural → high confidence

This prevents weak signals overpowering a strong contradiction.

---

## 7. Confidence Thresholds

| Confidence | Action              |
|----------:|---------------------|
| ≥ 0.90    | Accept automatically |
| 0.65–0.89 | Accept with warning  |
| 0.40–0.64 | Human review         |
| < 0.40    | Reject and retry     |

Thresholds should be tuned per field, not globally.

Identifiers require stricter thresholds than descriptive text.

---

## 8. Human-in-the-Loop Integration

Confidence scoring is not a reporting metric.
It is a workflow router.

Instead of reviewing everything:

Humans reviewing everything

> Humans review uncertainty only

This changes scaling from linear to selective verification.

---

## 9. Failure Modes Addressed

Confidence scoring specifically mitigates:

| Failure Type             | Example                             |
|--------------------------|-------------------------------------|
| Hallucinated identifiers | Invented company numbers            |
| OCR corruption           | 8 vs B errors                       |
| Context mis-binding      | Extracting example values           |
| Partial extraction       | Missing multi-field dependencies    |
| Silent regression        | Model update subtly changes output  |

---

## 10. Design Philosophy

We do not try to make the model perfect.

We make incorrect outputs detectable.

> Reliability comes from verification layers, not model intelligence.

---

## 11. Future Extensions

Planned additions to the open reference implementation:
- Calibration curves per field
- Confidence drift monitoring
- Adaptive thresholds based on document class
- Active learning from reviewer corrections

---

## Summary

Confidence scoring converts LLM output from an answer into a measurable claim.

Without confidence scoring:

> The system produces text.

With confidence scoring:

> The system produces evidence-weighted data.
