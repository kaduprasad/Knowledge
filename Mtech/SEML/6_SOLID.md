# Topic 6: Errors, Logging, Debugging, Idempotency, and SOLID

## Source and Priority

- Errors, logging, debugging: combined slides pages **90-92**.
- SOLID, modularity, refactoring: combined slides pages **95-96**.
- Pipeline idempotency: combined slides page **103**.
- September 2026 Q5 directly tests logging level and idempotency.

## 1. Error Handling

Error handling detects invalid conditions and prevents unpredictable pipeline behaviour.

```python
import logging


logger = logging.getLogger(__name__)


def prepare_age(age: int | None) -> int:
    if age is None:
        logger.warning("Age missing; using default value")
        return 40
    if age < 0:
        raise ValueError("Age cannot be negative")
    return age
```

- Missing age is recoverable, so it produces a warning and a documented fallback.
- Negative age is invalid, so the function fails early with an actionable exception.

### Professional Rules

1. Catch specific exceptions, not a bare `except:`.
2. Use **Failing Fast**: fail early before corrupted data moves downstream.
3. Use meaningful messages and domain-specific exceptions.
4. Log the failure; do not silently ignore it.
5. Return a default only when the fallback is valid and documented.

## 2. Common Exceptions

| Exception | ML/data example |
|---|---|
| `ValueError` | Negative age or invalid hyperparameter value |
| `KeyError` | Missing DataFrame column or dictionary key |
| `TypeError` | String supplied where a numerical array is required |
| `IndexError` | Accessing a row or feature position that does not exist |
| Custom exception | `ModelWeightsNotLoadedError` or `IncompatibleFeatureSizeError` |

## 3. Logging Levels

| Level | Use | Example |
|---|---|---|
| DEBUG | Detailed troubleshooting information | Input shape and intermediate values |
| INFO | Normal milestone | Model loaded successfully |
| WARNING | Unexpected but recoverable condition | Missing feature replaced by a valid default |
| ERROR | Operation failed | Prediction failed because the model is unavailable |
| CRITICAL | System/service cannot continue safely | Corrupted model registry or unavailable critical infrastructure |

Logging is better than print because it provides persistent history, severity categories, timestamps, module names, and line numbers.

## 4. Systematic Debugging

```text
Reproduce -> read traceback bottom-up -> isolate component
-> inspect state -> identify root cause -> fix -> regression test
```

Useful strategies:

- Reproduce the bug consistently.
- Read the last traceback line for the exception type/message.
- Use divide-and-conquer to isolate the failing block.
- Explain the code aloud using rubber-duck debugging.
- Step through execution with a debugger or IDE.
- Document the root cause and add a test.

## 5. Robustness

Robust code behaves predictably with missing, malformed, unusual, or corrupted input.

```python
def validate_features(features: dict[str, float]) -> None:
    required = {"age", "heart_rate", "oxygen"}
    missing = required - features.keys()
    if missing:
        raise ValueError(f"Missing features: {sorted(missing)}")
```

Robustness uses validation, exceptions, logging, safe fallback, tests, and monitoring.

## 6. Idempotency

An operation is idempotent when repeating it with the same input and unchanged configuration produces the same result without duplicate side effects.

```text
pipeline(input, config) = output
pipeline(input, config) = the same output again
```

Example:

```python
def normalize_age(age: float) -> float:
    return round(age / 100, 4)
```

Repeated calls with age 40 always return 0.4.

Non-idempotent example:

```python
def append_prediction(prediction: float, results: list[float]) -> None:
    results.append(prediction)
```

Repeating this operation duplicates the stored prediction. An idempotent storage operation would use a stable record ID and update/upsert the same record.

## Real Q5 Statements

### Statement A

“A missing input feature replaced with a default value should always be logged at ERROR level.”

**Disagree.** If the fallback is valid and processing continues, use WARNING. Use ERROR when the operation fails or cannot safely continue.

### Statement B

“Once a list is created, its elements cannot be modified.”

**Disagree.** Lists are mutable.

### Statement C

“An idempotent ML pipeline produces the same output when repeatedly executed with identical inputs and unchanged configuration.”

**Agree.** This is the definition of pipeline idempotency.

## 7. SOLID Principles

The page-95 lecture slide explicitly emphasizes Single Responsibility, Open/Closed, and Dependency Inversion. The complete standard SOLID acronym contains five principles:

| Letter | Principle | Meaning | ML example |
|---|---|---|---|
| S | Single Responsibility | One class/function should have one main responsibility | Separate data loading, preprocessing, training, and evaluation |
| O | Open/Closed | Extend behaviour without modifying stable existing code | Add a new model through a common interface |
| L | Liskov Substitution | A subtype must safely replace its base type | Any classifier implementation must honour the same `predict` contract |
| I | Interface Segregation | Prefer small focused interfaces over one large interface | Predictor clients should not be forced to implement training methods |
| D | Dependency Inversion | Depend on abstractions, not concrete implementations | Training code accepts a data-source interface instead of hard-coding SQL |

### Compact Example

```python
from typing import Protocol


class DataSource(Protocol):
    def load(self) -> list[dict]: ...


class TrainingPipeline:
    def __init__(self, source: DataSource):
        self.source = source

    def run(self) -> None:
        records = self.source.load()
        # validate, train, and evaluate through separate components
```

TrainingPipeline depends on the DataSource abstraction. SQL, CSV, or S3 implementations can be substituted without rewriting its main logic.

## Likely Exam Questions

1. Select the correct logging level for missing input, model-load failure, and normal milestones.
2. Explain fail-fast error handling in an ML pipeline.
3. Differentiate logging and print statements.
4. Explain idempotency and correct a duplicate-writing pipeline.
5. Describe a systematic debugging process for incorrect predictions.
6. State SOLID principles and apply S, O, and D to an ML pipeline.
7. Identify code smells: long function, duplicated logic, large class, and hard-coded configuration.

## One-Minute Recall

```text
Error handling -> fail predictably
Logging        -> preserve diagnostic evidence
Debugging      -> find and remove root cause
Robustness     -> handle abnormal input safely
Idempotency    -> same input/config, same result and no duplicate side effects

SOLID:
S -> one responsibility
O -> extend without modifying
L -> subtype can replace base type
I -> small focused interfaces
D -> depend on abstractions
```
