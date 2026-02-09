# ARIES_ML
# Complaint Classification System  
Primary • Secondary • Severity Prediction

## Overview

This project builds an end-to-end NLP system to classify customer complaints into:

1. Primary Category (broad product area)
2. Secondary Category (specific issue / root cause)
3. Severity (1–5 urgency score)

The system was designed explicitly around the evaluation metric and dataset constraints, prioritizing correctness, robustness, and defensible modeling decisions over leaderboard-specific hacks.

---

## Evaluation Metric

Final Score is computed as:

Final Score =  
0.3 × Primary Accuracy  
+ 0.4 × Secondary Accuracy  
+ 0.3 × Severity R²

Key implications from the metric:
- Secondary classification dominates overall performance.
- Severity is evaluated as a regression (R²), not classification accuracy.
- Exact label match matters more than semantic similarity.

---

## Data Observations and Early Insights

### Primary Categories
- Highly imbalanced.
- Credit-reporting-related categories dominate the dataset.
- Some primary categories have extremely small sample counts.

Insight:  
Primary labels are coarse and lexically separable. Simple linear models are sufficient.

---

### Secondary Categories
- Roughly balanced but semantically overlapping.
- Several near-duplicate labels with inconsistent human annotation, e.g.:
  - “Incorrect information on credit report”
  - “Incorrect information on your report”

Insight:  
Secondary classification is inherently ambiguous and annotation-noisy. This is the core bottleneck.

---

### Severity
- Extremely skewed distribution:
  - ~70% severity = 1
  - ~18–20% severity = 5
  - Very few samples in levels 2–4

Insight:  
Optimizing severity requires minimizing squared error, not producing balanced predictions.

---

## System Design Philosophy

A modular pipeline was adopted:

Complaint Text  
→ Primary Classification  
→ Secondary Classification  
→ Severity Estimation  
→ Submission Formatting

Each component was optimized independently and frozen once saturated. All decisions were driven by metric alignment rather than generic NLP best practices.

---

## Primary Classification

### Method Used
- TF-IDF features (word + phrase n-grams)
- Logistic Regression with strong regularization

### Rationale
- Strong lexical signal
- Robust under imbalance
- Interpretable and difficult to outperform with heavier models

### Internal Normalization
All credit-reporting-related primaries were merged internally into:

Credit reporting (all)

This reduced label sparsity and improved recall.

### Submission-Time Fix
At output time, the merged label was conditionally un-merged based on the predicted secondary category.

### Outcome
Primary accuracy saturated early and was not the limiting factor.

---

## Secondary Classification (Main Bottleneck)

Secondary classification accounts for 40% of the final score and proved to be the hardest component.

### Methods Explored

- TF-IDF + Logistic Regression (baseline)
- Hard Primary → Secondary constraints
- Embedding-based prototypes (mean)
- Clean prototypes (central samples only)
- Cross-encoder reranking
- Fine-tuned transformer per primary

### Results Summary

- TF-IDF baseline: ~0.60
- Constraints + embeddings: ~0.66
- Cross-encoder reranking: degraded performance
- Fine-tuned transformer (clean): best result

---

## Final Secondary Method

The final secondary classifier used a fine-tuned sentence transformer:

- Base model: all-MiniLM-L6-v2
- One model per primary category
- Cross-entropy loss
- Training limited to 2 epochs to avoid overfitting
- No ensembling
- No test feedback or leakage

### Why Per-Primary Fine-Tuning
- Secondary labels are conditional on primary
- Reduces class confusion
- Improves signal-to-noise ratio
- Learns dataset-specific annotation quirks

This produced the highest clean performance without exploiting evaluation artifacts.

---

## Severity Estimation

### Key Insight
Severity is evaluated using R², not accuracy. Incorrect mid-level predictions are heavily penalized.

### Final Strategy
- Rule-based detection for severity 5 (fraud, legal risk, security issues)
- Conservative fallback biased toward severity 1
- Mid-level severities (2–4) predicted only with strong evidence

### Outcome
- High Severity R²
- Skewed predictions toward 1 and 5 (intentional and optimal)

Severity was frozen early and was not a bottleneck.

---

## Leaderboard Experiments and Findings

Aggressive leaderboard strategies were tested to understand higher scores (0.75+):

- Regex-only secondary forcing
- Label collapsing with heuristic remapping
- Dominant-label biasing

### Empirical Findings
- Partial forcing → ~0.66
- Full forcing (100% regex hit rate) → score collapsed to ~0.61

Conclusion:
Leaderboard-level scores require iterative tuning using hidden test feedback or annotation artifact exploitation. These methods were intentionally rejected.

---

## Final Results

| Approach | Final Score |
|--------|-------------|
| TF-IDF baseline | ~0.60 |
| Embeddings + clean prototypes | ~0.66 |
| Fine-tuned transformer (clean) | 0.69784 |
| Aggressive leaderboard forcing | ~0.61 |

Final accepted score:

0.67904

This represents the true signal ceiling achievable without test leakage or feedback-driven tuning.

---

## Final System Summary

- Primary: TF-IDF + Logistic Regression with internal label merge
- Secondary: Fine-tuned MiniLM per primary (2 epochs)
- Severity: Conservative rule-based + regression model
- Submission: Conditional un-merge of credit-reporting labels

The system is production-grade, logically consistent, and defensible.

---

## Key Takeaways

- Metric alignment matters more than model complexity
- Secondary classification dominates overall performance
- Label ambiguity and annotation noise impose a real ceiling
- Leaderboard scores are not equivalent to model quality
- Knowing when to stop optimizing is a critical engineering skill

---

## Conclusion

The achieved score of 0.69784

This project demonstrates disciplined ML engineering, proper handling of noisy labels
