# PythonCore → Control Flow → `break / continue / else` (Loop Control)

# 1) Conceptual Semantics

## 1.1 `break`

Immediately terminates the nearest enclosing loop (`for` or `while`).

## 1.2 `continue`

Skips the rest of the current iteration and proceeds to the next
iteration (re-check condition for `while`, next item for `for`).

## 1.3 Loop `else`

Runs only if the loop ends **normally** (no `break`): - `for`: iterator
exhausted - `while`: condition becomes falsy

------------------------------------------------------------------------

# 2) CPython Runtime Mechanics (High Level)

These constructs compile to jumps.

## 2.1 `break` is a jump to loop end

But it also must "unwind" the loop's block stack (especially if
`try/finally` exists).

## 2.2 `continue` is a jump to loop header

-   `for`: jump to `FOR_ITER` fetch
-   `while`: jump to condition evaluation

## 2.3 `else` is positioned after the loop body

But is guarded so it runs only on normal loop exit, not break.

Implementation model: - Break jumps past the `else` block. - Normal exit
flows into `else`.

------------------------------------------------------------------------

# 3) `for ... else` and `while ... else` (Correct Mental Model)

Think of loop `else` as:

> "no-break handler"

### Search pattern:

``` python
for item in items:
    if matches(item):
        found = item
        break
else:
    found = None  # executed only if loop didn't break
```

This is equivalent to a flag, but clearer when used correctly.

------------------------------------------------------------------------

# 4) Edge Cases & Pitfalls

## 4.1 Nested loops: `break` only breaks one level

Common bug:

``` python
for row in rows:
    for col in cols:
        if bad:
            break  # only breaks inner loop
```

If you intend to break outer loop too: - use a flag - refactor into a
function and `return` - raise a controlled exception (rare, but
sometimes clean)

## 4.2 `continue` can skip critical work

Classic bug: skipping updates/metrics/logging/cleanup:

``` python
for x in xs:
    if not ok(x):
        continue
    update_metrics(x)  # might never run for invalid items
```

Make sure critical actions happen in the right place, or use
`try/finally` carefully.

## 4.3 `try/finally` interacts with break/continue

`finally` runs even when you `break` or `continue`.

``` python
for x in xs:
    try:
        ...
        if stop:
            break
    finally:
        cleanup()
```

Cleanup is guaranteed; this is a powerful pattern.

## 4.4 Loop `else` misunderstood

Many engineers misread loop `else` as "if loop didn't run". It's not.

`else` runs if: - loop ran zero iterations AND no break occurred (still
normal exit) - OR loop ran and completed without break

So it is truly "no break occurred".

------------------------------------------------------------------------

# 5) Performance Considerations

-   `break` can be a big win because it avoids additional iterations
-   `continue` can reduce work per iteration but may obscure logic
-   Loop `else` has negligible overhead; it's about
    correctness/readability

------------------------------------------------------------------------

# 6) Production Patterns (Backend Grade)

## 6.1 Early exit search + else

Used for scans:

-   find first matching record
-   locate free slot
-   validate constraints

## 6.2 Validation loops

Example: ensure no duplicates:

``` python
seen = set()
for x in items:
    if x in seen:
        raise ValueError("duplicate")
    seen.add(x)
else:
    return True
```

## 6.3 Retry loop with break on success

Often paired with `while ... else` for failure handler:

``` python
attempts = 3
while attempts:
    if try_once():
        break
    attempts -= 1
else:
    raise RuntimeError("all retries failed")
```

------------------------------------------------------------------------

# 7) Interview Traps

1)  Loop `else` runs only if loop wasn't broken
2)  `break` breaks only the nearest loop
3)  `continue` skips to next iteration and may skip critical code
4)  `finally` executes even on break/continue
5)  Search-with-else is a canonical Python idiom

------------------------------------------------------------------------

# 8) Summary

-   `break` = terminate loop, skip loop-else
-   `continue` = jump to next iteration
-   loop `else` = "no break happened" handler
-   Correctness depends on clear mental model + careful nesting

------------------------------------------------------------------------

