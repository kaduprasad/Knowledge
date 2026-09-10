# Topic 7: Quality Assurance for ML Systems

## Source and Priority

- Combined SEML slides: [`SEML_all_paginated_compressed.pdf`](../../materials/lecture/SEML_all_paginated_compressed.pdf), pages **100-108**.
- Priority: **Tier 2, high-probability design, comparison, metric, or scenario question**.
- No standalone QA question appears in the supplied September 2026 EC3 paper, but idempotency, monitoring, RAG, and security connect directly to Q3, Q5, Q6, and Q7.

![ML quality assurance stack](../assets/seml_ml_quality_assurance_stack.png)

## Core Idea

Traditional software tests verify explicitly programmed logic. ML quality assurance must additionally verify data, learning behaviour, statistical outputs, serving infrastructure, and changing production conditions.

```text
Data quality -> Pipeline quality -> Model quality
-> Serving/system quality -> Production monitoring and feedback
```

## 1. Traditional Software Testing Versus ML Testing

| Traditional software | ML system |
|---|---|
| Logic is explicitly programmed | Behaviour is learned from data |
| Tests compare deterministic input/output | Tests include statistical thresholds and distributions |
| Common failures: coding defects | Additional failures: bad data, bias, drift, overfitting |
| Main artifact: code | Artifacts: code, data, features, model, configuration |
| Same input usually gives expected output | Quality may vary across datasets and subgroups |

## 2. Types of ML Tests

| Test type | What it verifies | Example |
|---|---|---|
| Unit test | One function/component | Normalization returns the expected value |
| Integration test | Components work together | Preprocessor output is accepted by the model |
| End-to-end test | Complete user workflow | API request produces a valid prediction response |
| Data test | Schema, missing values, ranges, distributions | Required age column exists and age is non-negative |
| Model test | Learning and statistical quality | F1-score exceeds the release threshold |
| Infrastructure test | Serving capacity and reliability | API latency under concurrent load |
| Heuristic test | Domain/common-sense rule | Probability lies between 0 and 1 |
| Fairness test | Quality across subgroups | Recall is checked separately by demographic group |

## 3. Data Quality

Garbage in, garbage out: data quality sets the upper limit on model quality.

| Dimension | Meaning | Example |
|---|---|---|
| Accuracy | Data reflects reality | Churn labels are correct |
| Completeness | Required values are present | Every patient record has age and oxygen level |
| Consistency | Formats and meaning agree across systems | All dates use YYYY-MM-DD |
| Timeliness | Data is sufficiently current | Stock prices are not delayed |

### Data Validation Pipeline

```text
Ingestion
-> Schema gate
-> Value/range gate
-> Label-health gate
-> Pass: continue
-> Fail: halt, alert, and quarantine
```

- Schema gate detects missing columns and changed types.
- Value gate detects impossible values and statistical outliers.
- Label gate detects missing labels and extreme class imbalance.
- Invalid data should not silently reach training or serving.

## 4. Pipeline Quality

Pipeline quality is the robustness and reproducibility of the automated path from extraction to deployment.

Key requirements:

- Idempotency: identical inputs and configuration produce the same output.
- Versioning: code, data, model, and configuration are linked.
- Orchestration: dependencies and execution order are managed.
- Observability: stages expose logs, metrics, status, and failure information.
- Recovery: failed stages can retry or resume without corrupting results.

### Continuous Integration for ML

```text
Code change -> syntax/unit tests
Model architecture change -> small synthetic training test
Data schema change -> compatibility/validation tests
All checks pass -> deployable artifact in registry
```

## 5. Model Quality

Accuracy alone may be misleading, especially with imbalanced data. Model quality also includes fairness, interpretability, robustness, computational efficiency, and performance on unseen data.

### Classification Metrics

Precision asks: Of all predicted positives, how many were correct?

$$
\text{Precision}=\frac{TP}{TP+FP}
$$

Recall asks: Of all actual positives, how many were found?

$$
\text{Recall}=\frac{TP}{TP+FN}
$$

F1-score balances precision and recall.

$$
F_1=2\cdot\frac{\text{Precision}\cdot\text{Recall}}
{\text{Precision}+\text{Recall}}
$$

Where:

- TP = true positives.
- FP = false positives.
- FN = false negatives.

### Regression Metric

RMSE gives larger errors more weight.

$$
\text{RMSE}=\sqrt{\frac{1}{n}\sum_{i=1}^{n}(y_i-\hat{y}_i)^2}
$$

- $n$ = number of examples.
- $y_i$ = actual value.
- $\hat{y}_i$ = predicted value.

## 6. Slicing and Subpopulation Testing

A model may show good global accuracy while failing badly for one group.

Example:

```text
Global accuracy: 95%
Desktop users: 97%
Mobile minority subgroup: 50%
```

Procedure:

1. Divide test data by gender, location, device, age group, or another relevant dimension.
2. Calculate metrics separately for each slice.
3. Compare disparities against acceptance thresholds.
4. Investigate data coverage, labels, features, and model behaviour.
5. Mitigate and retrain where necessary.

## 7. Training Testing Versus Inference Testing

| Training testing | Inference testing |
|---|---|
| Verifies learning and convergence | Verifies prediction behaviour and serving |
| Checks loss, overfitting, gradients, and reproducibility | Checks latency, output format, stability, and concurrency |
| Long-running, high-throughput batch compute | Low-latency, highly available online service |
| Failures: exploding gradients, OOM, failure to converge | Failures: API timeout, invalid shape, unavailable model |

## 8. System Quality and Telemetry

The model is only one part of the application. A highly accurate model has no business value if the API, database, network, or user interface fails.

| Metric category | Examples |
|---|---|
| System metrics | CPU, memory, network latency, error rate |
| Model metrics | Prediction distribution, confidence, anomalies, drift |
| Business metrics | Conversion, click-through rate, revenue, clinical outcome |
| Alerts | Threshold violations that notify responsible teams |

## 9. Production Experimentation

Offline data is static, while users and environments change. Production experiments provide evidence of real-world behaviour and business impact.

| Method | Traffic behaviour | User sees new output? | Main purpose |
|---|---|---|---|
| A/B test | Users are split between versions | Yes, assigned version | Compare user/business outcomes |
| Shadow deployment | Requests are copied to new version | No | Validate safely using real traffic |
| Canary rollout | Small percentage receives new version | Yes, small group | Detect risk before full rollout |

## 10. ML Security Across the Lifecycle

| Lifecycle phase | Threat | Example control |
|---|---|---|
| Data collection | Data poisoning | Validate sources, ranges, labels, and anomalies |
| Model training | Backdoor attack | Trusted code/data, reproducible training, model evaluation |
| Deployment | Supply-chain attack | Pin and scan dependencies/images; verify artifacts |
| Inference | Adversarial input or extraction | Input validation, rate limits, access control, monitoring |

Security must protect training data, model intellectual property, infrastructure, and private user information.

## 11. RAGAS for RAG Quality

RAGAS means Retrieval-Augmented Generation Assessment. It evaluates retriever and generator quality, often using an LLM evaluator without requiring human labels for every example.

### Two RAG Pipelines

```text
Offline indexing: documents -> chunks -> embeddings -> vector database
Online query: query -> retrieval -> context -> LLM -> answer
```

### Core RAGAS Metrics

| Metric | Component | Question answered |
|---|---|---|
| Faithfulness | Generator | Is the answer supported by retrieved context? |
| Answer relevance | Generator | Does the answer address the user's question? |
| Context precision | Retriever | Are retrieved chunks relevant rather than noise? |
| Context recall | Retriever | Were all necessary facts retrieved? |

Memory rule:

```text
High context precision -> little irrelevant context
High context recall    -> few required facts missing
High faithfulness      -> little unsupported/hallucinated content
High answer relevance  -> directly answers the question
```

## High-Probability Exam Questions

### 1. Design QA for a Patient-Risk System, 6 Marks

Include schema/range tests, model thresholds, subgroup tests, inference latency, monitoring, alerting, and safe deployment.

### 2. Traditional Software Versus ML Testing, 4 Marks

Compare programmed versus learned behaviour, deterministic versus statistical tests, and code versus data/model failure modes.

### 3. Data Validation Pipeline, 6 Marks

Draw ingestion, schema, value, label-health, and action gates; explain halt/quarantine on failure.

### 4. Global Accuracy Trap, 4 Marks

Explain why 95% global accuracy can hide 50% performance for a minority group and how slicing reveals it.

### 5. Training Versus Inference Testing, 4 Marks

Compare purpose, compute constraints, failure modes, and environment.

### 6. A/B Versus Shadow Versus Canary, 6 Marks

Compare traffic routing, user exposure, purpose, risk, and suitable use case.

### 7. Lifecycle Security, 6 Marks

Map data poisoning, backdoors, supply-chain attacks, and adversarial inference to lifecycle stages and controls.

### 8. RAGAS Metrics, 6 Marks

Differentiate faithfulness, answer relevance, context precision, and context recall and identify whether each evaluates retrieval or generation.

## Common Traps

1. High accuracy proves the model is good for every group: **Incorrect; test slices separately.**
2. Unit tests alone assure ML quality: **Incorrect; also test data, model, pipeline, infrastructure, and production behaviour.**
3. Readiness and low latency prove prediction quality: **Incorrect; they measure serving health, not statistical correctness.**
4. Shadow deployment affects user decisions: **Incorrect; shadow output is not returned to users.**
5. Canary and A/B testing have the same objective: **Incorrect; canary limits release risk, while A/B compares outcomes.**
6. Context precision and faithfulness are identical: **Incorrect; precision evaluates retrieval, faithfulness evaluates grounding of the generated answer.**

## One-Minute Recall

```text
Data QA       -> correct, complete, consistent, timely
Pipeline QA   -> reproducible, idempotent, versioned, observable
Model QA      -> suitable metrics, unseen data, subgroup slices
Inference QA  -> format, latency, stability, concurrency
Production QA -> telemetry, A/B, shadow, canary
Security QA   -> threats across the ML lifecycle
RAG QA        -> retrieval quality plus grounded generation
```
