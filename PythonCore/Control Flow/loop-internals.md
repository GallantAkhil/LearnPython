# PythonCore → Control Flow → `for` Loop Internals

# 1) Conceptual Model

Python `for` is **not** index-based looping. It is **iterator-driven**.

``` python
for x in iterable:
    body
```

Semantics:

1)  Obtain an iterator: `it = iter(iterable)`
2)  Repeatedly fetch next: `x = next(it)`
3)  Execute body
4)  Stop when `StopIteration` is raised

This is the core invariant across builtins and user-defined iterables.

------------------------------------------------------------------------

# 2) The Iteration Protocol

## 2.1 Iterable vs Iterator

-   Iterable: has `__iter__()` returning an iterator
-   Iterator: has `__iter__()` returning self AND `__next__()`

``` python
iterable.__iter__() -> iterator
iterator.__next__() -> item or raises StopIteration
```

## 2.2 Fallback protocol (`__getitem__`)

If an object lacks `__iter__` but has `__getitem__` with integer indexes
starting at 0, `iter(obj)` can still work.

This matters for legacy containers and custom sequences.

Pitfall: `__getitem__` iteration is slower and can hide performance
problems.

------------------------------------------------------------------------

# 3) CPython Compilation Model (Bytecode)

`for` compiles to a loop driven by `FOR_ITER`.

High-level bytecode shape:

1)  Evaluate iterable expression
2)  `GET_ITER` → obtain iterator
3)  Loop:
    -   `FOR_ITER` → fetch next item, jump out on `StopIteration`
    -   Store into loop target
    -   Execute body
    -   Jump back to `FOR_ITER`

Conceptual skeleton:

    SETUP_LOOP end
      <push iterable>
      GET_ITER
    start:
      FOR_ITER end
      STORE_FAST target
      <body>
      JUMP_ABSOLUTE start
    end:
    POP_BLOCK

Notes: - Exact opcodes change across 3.10--3.12+, but the control
structure stays the same. - `FOR_ITER` is a specialized opcode that
handles `StopIteration` control flow without exposing exception
machinery to Python level each time.

------------------------------------------------------------------------

# 4) Runtime Mechanics (What Actually Happens)

## 4.1 `StopIteration` is control flow

Iterator signals termination by raising `StopIteration`. The loop
catches it internally.

Important: - If your `__next__` raises `StopIteration` incorrectly
early, loop truncates. - If your `__next__` raises
`StopIteration(value)`, the value is ignored by `for`.

## 4.2 Exception propagation from the body

Any exception inside the loop body immediately exits the loop and
propagates outward (unless caught).

## 4.3 Mutation during iteration

Mutating the underlying container has type-specific outcomes:

-   `list`: mutation can lead to skipped or repeated items (no
    fail-fast)
-   `dict`/`set`: mutation during iteration raises `RuntimeError`
    ("changed size during iteration")

Engineering rule: \> Don't mutate the container you're iterating unless
you fully understand the container's iteration semantics.

------------------------------------------------------------------------

# 5) Specialized Optimizations in CPython

CPython has fast paths for common iterators, especially:

-   List iterator
-   Tuple iterator
-   Range iterator
-   Dict key/value/item iterators

These iterators are C structs with minimal overhead per step.

### `range` iteration is especially efficient

-   Produces ints without allocating a list
-   Each step is arithmetic

### `enumerate` and `zip` are streaming (no materialization)

-   Very useful for backend pipelines and memory efficiency

------------------------------------------------------------------------

# 6) Edge Cases & Pitfalls

## 6.1 Iterating over a generator that raises non-StopIteration

If generator raises an exception, loop stops and exception propagates.

## 6.2 Consuming iterators accidentally

Iterators are stateful:

``` python
it = iter([1,2,3])
list(it)  # consumes
list(it)  # []
```

In backend code, this can cause "empty second pass" bugs.

## 6.3 Iterating over bytes vs str

-   `for b in b"abc"` yields ints (97, 98, 99)
-   `for ch in "abc"` yields 1-length strings

## 6.4 `next(it, default)` vs try/except StopIteration

The 2-arg `next` avoids exception overhead in control logic.

------------------------------------------------------------------------

# 7) Performance Considerations

## 7.1 Python loop overhead dominates

In tight loops, overhead is: - bytecode dispatch - dynamic type checks -
reference counting

Not CPU-level branch prediction.

## 7.2 Prefer builtins that run in C

When possible: - `sum`, `min`, `max`, `any`, `all`, `sorted` -
`''.join(...)` - `collections.Counter`, `itertools` utilities

These reduce Python-level per-iteration overhead.

## 7.3 Avoid repeated global lookups

Bind frequently used functions locally:

``` python
append = out.append
for x in data:
    append(f(x))
```

This reduces LOAD_GLOBAL overhead in hot loops.

## 7.4 Materialization vs streaming

-   Streaming iterators are memory efficient
-   But sometimes repeated passes require materialization (list) for
    correctness

Make the trade consciously.

------------------------------------------------------------------------

# 8) Backend Engineering Patterns

## 8.1 Guarded streaming pipeline

Process large data without loading all into memory:

``` python
for row in stream_rows():
    if row.is_bad:
        continue
    process(row)
```

## 8.2 Batching (chunking)

Avoid per-item DB commits; batch work:

``` python
batch = []
for item in items:
    batch.append(item)
    if len(batch) == 1000:
        flush(batch)
        batch.clear()
if batch:
    flush(batch)
```

## 8.3 Early exit with `break`

Use when you can stop as soon as you find the answer:

``` python
for u in users:
    if u.id == target:
        found = u
        break
else:
    found = None
```

(Loop `else` explained in its own document, but it pairs naturally with
`for`.)

------------------------------------------------------------------------

# 9) Interview-Level Traps

1)  `for` is iterator-based, not index-based
2)  `StopIteration` terminates loops (internal control flow)
3)  Iterators are single-pass; iterables can be multi-pass
4)  `dict` iteration fails on size change; `list` doesn't
5)  `range` is lazy and efficient
6)  `bytes` iteration yields ints

------------------------------------------------------------------------

# 10) Summary

-   `for` uses `iter()` + `next()` + `StopIteration`
-   CPython uses `GET_ITER` + `FOR_ITER` bytecode for efficient
    iteration
-   Builtin iterators have optimized C implementations
-   Performance depends more on Python overhead than on the loop
    construct itself
-   For backend systems: use streaming, batching, and C-level builtins
    for speed

------------------------------------------------------------------------
