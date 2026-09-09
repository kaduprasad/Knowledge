# Topic 12: Python Timing and Profilers

## Source and Priority

- Combined SEML slides: [`SEML_all_paginated_compressed.pdf`](../../materials/lecture/SEML_all_paginated_compressed.pdf), pages **75-80**.
- Priority: **Tier 3, high-probability short theory, comparison, output-interpretation, or tool-selection question**.
- No standalone profiling question appears in the supplied September 2026 EC3 paper.

## Why Profile Code?

Timing tells us that a program is slow. Profiling tells us **where** it spends time or memory.

```text
Measure -> find bottleneck -> optimize that bottleneck -> measure again
```

Do not optimize every line before measuring. The lecture explicitly warns against **premature optimization**.

## Tool-Selection Table

| Tool | What it measures | Detail level | Best question answered |
|---|---|---|---|
| `time` | Elapsed time for one execution | Code block/workflow | How long did this run take? |
| `timeit` / `%%timeit` | Repeated execution time | Small function/snippet | Which implementation is consistently faster? |
| `cProfile` | Function calls and CPU execution time | Function level | Which function is the bottleneck? |
| `line_profiler` | Time spent on each source line | Line level | Which line inside the slow function is expensive? |
| `memory_profiler` | Memory usage and increment per line | Line level | Which line increases memory usage? |
| Memray | Allocations, memory usage, and leaks | Allocation/call level | Which function allocates memory or leaks it? |

The lecture notes state that **Memray supports macOS/Linux, not Windows**. On Windows, use `memory_profiler` for the taught line-by-line memory view.

## 1. `time`: Measure One Complete Run

```python
import time

start = time.time()
model.fit(x_train, y_train)
end = time.time()

print(f"Training time: {end - start:.6f} seconds")
```

Use it for one complete workflow, such as loading data, training, or prediction. A single result can be affected by CPU load, caching, and background processes.

## 2. `timeit`: Repeated Benchmarking

```python
import statistics
import timeit

totals = timeit.repeat(
    "sum(range(10_000))",
    repeat=10,
    number=100,
)
average_per_run = [total / 100 for total in totals]

print(statistics.mean(average_per_run))
print(statistics.stdev(average_per_run))
```

- `number=100`: execute the statement 100 times in each benchmark.
- `repeat=10`: repeat that benchmark ten times.
- Divide each total by 100 to obtain time per execution.
- A small standard deviation means performance is stable across runs.

In Jupyter or Colab:

```python
%%timeit
model.fit(x_train, y_train)
```

## `time` Versus `timeit`

| `time` | `timeit` |
|---|---|
| Usually measures one execution | Automatically runs code many times |
| Suitable for an entire workflow | Suitable for a small snippet or function |
| More sensitive to temporary system noise | Uses repeated runs for a more reliable comparison |
| Gives one elapsed duration | Can produce an average and standard deviation |

## 3. `cProfile`: Find the Slow Function

Example program: [`profiler_demo.py`](../examples/profiling/profiler_demo.py)

```powershell
python -m cProfile -s cumulative profiler_demo.py
```

`cProfile` is built into Python. It records each function's call count and execution time.

## Important Output Columns

| Column | Meaning |
|---|---|
| `ncalls` | Number of function calls |
| `tottime` | Time spent inside the function itself, excluding subfunctions |
| First `percall` | `tottime / ncalls` |
| `cumtime` | Time in the function plus all functions it called |
| Second `percall` | `cumtime / primitive calls` |
| `filename:lineno(function)` | Function source location and name |

Sort by `cumulative` when looking for the workflow/function responsible for most total time. Sort by `tottime` when looking for code that itself consumes CPU rather than delegating to subfunctions.

## Verified Example Result

On the local validation run:

| Function | Cumulative time |
|---|---:|
| `run_pipeline()` | About 0.822 s |
| `pairwise_similarity()` | About 0.821 s |
| Generator expression inside similarity | About 0.379 s |
| `load_records()` | About 0.001 s |

Therefore, optimize `pairwise_similarity()` first. Optimizing data loading would have almost no effect on total runtime.

Exact timing varies by machine; the ranking of this deterministic example is the important profiling result.

## 4. `line_profiler`: Find the Slow Line

After `cProfile` identifies a slow function, inspect its individual lines.

```python
@profile
def pairwise_similarity(records):
    total_distance = 0.0
    for index, current in enumerate(records):
        for other in records[index + 1:]:
            total_distance += sum(
                abs(left - right)
                for left, right in zip(current, other)
            )
    return total_distance
```

Typical commands:

```powershell
pip install line_profiler
kernprof -l -v profiler_demo.py
```

Important columns usually include line number, hits, total time, time per hit, percentage of total time, and line contents.

## `cProfile` Versus `line_profiler`

| `cProfile` | `line_profiler` |
|---|---|
| Profiles functions | Profiles individual lines |
| Finds which function is slow | Finds which line inside that function is slow |
| Good first profiler for a complete script | Use after narrowing the search to a function |
| Output may include many internal library calls | Output maps directly to source lines |

## 5. Memory Profiling

## Windows: `memory_profiler`

```python
from memory_profiler import profile


@profile
def build_feature_matrix():
    matrix = [[0.0] * 1_000 for _ in range(10_000)]
    return matrix


build_feature_matrix()
```

```powershell
pip install memory_profiler
python -m memory_profiler memory_example.py
```

Important columns:

| Column | Meaning |
|---|---|
| `Mem usage` | Total process memory at that line |
| `Increment` | Additional memory caused by that line |
| `Occurrences` | Number of times the line executed |
| `Line Contents` | Source statement being measured |

A large positive `Increment` identifies a line that allocates substantial memory.

## macOS/Linux: Memray

Memray helps answer:

- Which function allocates the most memory?
- How much memory is allocated?
- Where are memory leaks occurring?
- Which allocation path retained the memory?

It is a memory profiler, not a replacement for `cProfile`, which measures execution time.

## Recommended Profiling Workflow

1. Measure one end-to-end run with `time`.
2. Use `timeit` when comparing small alternative implementations.
3. Run `cProfile` to identify the expensive function.
4. Use `line_profiler` on that function to identify expensive lines.
5. Use `memory_profiler` or Memray when memory growth is the problem.
6. Optimize only the measured bottleneck.
7. Rerun the same benchmark to prove improvement.

## Likely Exam-Style Questions

## 1. Compare Timing and Profiling Tools, 5 Marks

Differentiate `time`, `timeit`, `cProfile`, `line_profiler`, and a memory profiler by purpose and granularity.

## 2. Select the Correct Tool, 4 Marks

- Measure one complete training run -> `time`.
- Compare two preprocessing functions reliably -> `timeit`.
- Find the slow function in a script -> `cProfile`.
- Find the slow line inside `train_model()` -> `line_profiler`.
- Find the line allocating a large array -> `memory_profiler`.

## 3. Interpret `cProfile` Output, 4 Marks

Explain `ncalls`, `tottime`, and `cumtime`, then identify the first function that should be optimized.

## 4. Interpret Memory Output, 4 Marks

Explain `Mem usage`, `Increment`, and `Occurrences`, then identify the line responsible for the largest allocation.

## 5. Performance-Debugging Scenario, 6 Marks

An ML pipeline is slow after feature engineering is added. Describe the measurement sequence, identify whether CPU time or memory is the problem, optimize the measured hotspot, and verify the improvement with the same benchmark.

## Common Traps

1. **One `time` result proves an optimization:** Incorrect; repeated measurements are more reliable.
2. **`cProfile` identifies the exact slow line:** Incorrect; it primarily identifies slow functions.
3. **Highest `ncalls` always means bottleneck:** Incorrect; examine time, not call count alone.
4. **`tottime` and `cumtime` are identical:** Incorrect; cumulative time includes subfunction calls.
5. **Memory and CPU profiling are interchangeable:** Incorrect; they answer different questions.
6. **Optimize loading because it appears first:** Incorrect; optimize the measured hotspot, not execution order.

## One-Minute Recall

```text
time            -> one elapsed run
timeit          -> repeated benchmark
cProfile        -> slow function
line_profiler   -> slow line
memory profiler -> large allocation or leak
```
