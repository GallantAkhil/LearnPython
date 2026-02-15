# PythonCore → Functions (Advanced) → `lambda`

# 1) Conceptual Model

`lambda` creates a function object:

``` python
f = lambda x: x + 1
```

It is equivalent to:

``` python
def f(x):
    return x + 1
```

Differences: - `lambda` must be a single expression (no statements) -
name is `<lambda>` which impacts tracebacks/debugging - generally used
for short callbacks

------------------------------------------------------------------------

# 2) CPython Compilation

A lambda creates: - a `code` object (like any function) - a function
object referencing that code object - possibly closure cells if it
captures free variables

You can inspect: - `f.__code__.co_varnames` - `f.__code__.co_freevars` -
`f.__closure__`

------------------------------------------------------------------------

# 3) Closure + Late Binding

Lambda participates in closures exactly like `def`.

The classic bug:

``` python
funcs = [lambda: i for i in range(3)]
[f() for f in funcs]  # [2,2,2]
```

Fix:

``` python
funcs = [lambda i=i: i for i in range(3)]
```

------------------------------------------------------------------------

# 4) Performance Notes

Lambda has no special runtime speed advantage over `def`. It still
creates a function object and incurs normal call overhead.

In hot loops: - avoid creating lambdas repeatedly - hoist function
creation out of loops

------------------------------------------------------------------------

# 5) Production Guidance

## 5.1 Good uses

-   `key=` functions for sorting
-   small transformations in `map`/`filter` (but list comprehensions
    often clearer)
-   callbacks in frameworks when logic is trivial

## 5.2 Avoid when

-   logic spans multiple steps
-   you need logging/try/except/early returns
-   you want readable stack traces
-   you need proper naming and documentation

Prefer `def` for maintainability.

------------------------------------------------------------------------

# 6) Interview Traps

1)  Lambda is expression-only (no statements)
2)  Lambda participates in closures → late binding applies
3)  Lambda is not faster than `def`
4)  Debuggability suffers due to `<lambda>` name

------------------------------------------------------------------------

# 7) Summary

Lambda is a compact function literal with the same underlying machinery
as `def`. Use it for small, clear, local transformations; avoid it for
complex production logic.

------------------------------------------------------------------------
