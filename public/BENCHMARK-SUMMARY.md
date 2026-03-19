---
owner: product-team
audience: public
source_of_truth: true
status: active
repo: lexe-docs
review_cadence: monthly
sensitivity: public
last_verified: 2026-03-19
---

# LEXe Legal AI - Model Benchmark Summary

## Overview

LEXe conducts rigorous, reproducible benchmarks to select the best language models for legal research. Our evaluation methodology is designed to eliminate bias and measure what matters for legal professionals.

## Methodology

### Evaluation Framework

We use **G-Eval**, a reference-free evaluation protocol where independent judge models score responses across five dimensions specifically calibrated for Italian legal work:

| Dimension | What It Measures |
|-----------|-----------------|
| **Accuracy** | Correctness of legal citations, article references, and factual claims |
| **Completeness** | Coverage of relevant norms, jurisprudence, and doctrine |
| **Citations** | Proper attribution, verifiable source references, and link quality |
| **Italian Language** | Legal Italian fluency, proper terminology, register appropriateness |
| **Structure** | Logical organization, clear argumentation, professional formatting |

### Bias Controls

- **Independent judge model**: An external model (not a candidate) evaluates all responses
- **Peer-review validation**: A second judge cross-checks scores to detect self-preferencing or systematic bias
- **Blind evaluation**: Judge models receive anonymized responses without model identification
- **Multiple runs**: Six independent evaluation runs to ensure statistical stability

### Scale

- **11 candidate models** tested across multiple providers
- **6 evaluation runs** per model-task combination
- **2 independent judges** with cross-validation
- **Role-specific scoring weights** tailored to each system function (fast triage, deep research, verification, synthesis)

## Production Configuration

Based on benchmark results, LEXe uses **role-specific model assignments** where each system function (fast lookup, primary research, complex analysis, verification, frontier synthesis) is mapped to the model that scored highest for that role's weighted dimensions.

The current production model was selected for its consistent performance across all five evaluation dimensions, with particular strength in citation accuracy and Italian legal terminology.

## Role-Specific Optimization

Different system functions have different quality requirements:

- **Fast triage**: Prioritizes speed and accuracy for simple lookups
- **Primary research**: Balances completeness and citation quality
- **Complex analysis**: Maximizes completeness and structure for multi-source research
- **Verification**: Focuses on accuracy and citation correctness
- **Frontier synthesis**: Prioritizes structure, language quality, and argumentation depth

Each role applies different scoring weights during benchmarking to select the optimal model for that function.

## Continuous Evaluation

Benchmarks are re-run when new models become available or when existing models receive significant updates. Results are tracked over time to detect quality regressions.

---

*Model selection at LEXe is data-driven, bias-controlled, and continuously validated.*
