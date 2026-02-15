# PythonCore → Control Flow → `while` Loops

# 1) Conceptual Model

``` python
while condition:
    body
else:
    else_block
```

Semantics:

1)  Evaluate condition
2)  If falsy → exit loop, optionally run `else` block
3)  If truthy → execute body, then repeat

Key difference from `for`: - `while` is condition-driven (you manage
progress) - Easy to create infinite loops if you don't ensure progress

------------------------------------------------------------------------

# 2) CPython Compilation Model (Bytecode Shape)

High-level bytecode pattern:

-   Jump to condition test
-   If false: jump out
-   Execute body
-   Jump back to test

Conceptually:

    start:
      <evaluate condition>
      POP_JUMP_IF_FALSE end
      <body>
      JUMP_ABSOLUTE start
    end:
      <optional else>

The condition evaluation uses the full truthiness protocol
(`PyObject_IsTrue`).

------------------------------------------------------------------------

# 3) Termination Correctness (Engineering View)

## 3.1 Progress invariant

A correct while loop must satisfy:

-   There is a state variable that changes each iteration
-   The state moves toward making the condition false

Example progress variable:

``` python
i = 0
while i < n:
    ...
    i += 1
```

## 3.2 Timeouts and retry budgets

In backend systems, "eventually" often means "within budget". Use:

-   max retries
-   deadline timestamps
-   exponential backoff
-   circuit breaker semantics

Avoid unbounded polling loops.

------------------------------------------------------------------------

# 4) Common Backend Use-Cases

## 4.1 Retry loops with exponential backoff

Pattern:

-   Retry on transient failures
-   Backoff
-   Stop after budget

## 4.2 Polling for a condition

Polling is acceptable when: - you have backoff - you have deadline - you
can tolerate stale checks

## 4.3 Streaming sentinel loops

Classic pattern:

``` python
while (chunk := stream.read(8192)):
    process(chunk)
```

This is safe because empty bytes is falsy and indicates EOF.

------------------------------------------------------------------------

# 5) Edge Cases & Pitfalls

## 5.1 Condition recomputation cost

If condition is expensive, you pay it each iteration.

Hoist invariant parts:

``` python
deadline = time.time() + 5
while time.time() < deadline:
    ...
```

Even here, `time.time()` still called each iteration; but deadline is
hoisted.

## 5.2 Floating-point drift loops

Avoid conditions like:

``` python
x = 0.0
while x != 1.0:
    x += 0.1
```

This can become infinite due to floating representation error.

Use epsilon comparisons or integer counters.

## 5.3 `while True` without guaranteed break

`while True:` is fine only if: - every path either breaks or raises -
and you can prove it

## 5.4 Mutation-based termination hazards

If loop depends on external state (queue size, dict keys) and you mutate
incorrectly, you can create: - livelocks (loop runs but makes no
progress) - starvation (never reaches termination)

------------------------------------------------------------------------

# 6) `while ... else` Semantics

`else` executes only if the loop exits **normally** (condition becomes
false), not via `break`.

``` python
while cond:
    if found:
        break
else:
    # executed only if not broken
    handle_not_found()
```

This is most useful for "search until found" patterns.

------------------------------------------------------------------------

# 7) Performance Considerations

-   Python `while` loops have similar overhead to `for`
-   Use builtins/C-level functions inside loops where possible
-   Bind frequently-used functions locally in hot loops
-   Avoid repeated attribute/global lookups

Most speedups come from reducing Python-level work per iteration, not
from choosing while vs for.

------------------------------------------------------------------------

# 8) Interview Traps

1)  `while ... else` runs only if no `break`
2)  `while` condition uses truthiness protocol (not only bool)
3)  Infinite loops often come from missing progress invariants
4)  Float equality loops are unsafe
5)  `while True` must be proven to break

------------------------------------------------------------------------

# 9) Summary

-   `while` is condition-driven; you must enforce progress and budgets
-   CPython compiles it into conditional jumps around a loop block
-   `else` is a "no break occurred" handler
-   For backend code: always add budgets (timeout/retry cap) and ensure
    progress

------------------------------------------------------------------------

