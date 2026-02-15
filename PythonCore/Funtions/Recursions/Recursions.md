# PythonCore → Functions (Advanced) → Recursion

# 1) Conceptual Model

Recursion is function self-invocation that reduces a problem until a
base case.

Correct recursive design requires: - base case - progress (strictly
moving toward base case)

------------------------------------------------------------------------

# 2) CPython Call Stack & Frames

Every Python function call generally creates a frame containing: - local
variables - evaluation stack - instruction pointer - references to
globals/builtins

Recursive calls allocate many frames → memory growth.

------------------------------------------------------------------------

# 3) Recursion Limits

CPython enforces a recursion limit to prevent C stack overflow:

``` python
import sys
sys.getrecursionlimit()
sys.setrecursionlimit(n)
```

Raising the limit is dangerous: - Python frames still consume memory -
underlying C stack can overflow catastrophically

Production rule: \> Do not raise recursion limit in server code unless
you fully control recursion depth and have load-tested.

------------------------------------------------------------------------

# 4) Tail Recursion: No TCO in Python

Python does not perform tail-call optimization (TCO) by design.

Reasons: - debuggability (stack traces) - introspection - predictable
semantics

So this will still blow recursion limit:

``` python
def f(n, acc=0):
    if n == 0:
        return acc
    return f(n-1, acc+n)
```

Prefer iteration for deep recursion.

------------------------------------------------------------------------

# 5) Performance

Recursion overhead includes: - frame creation - argument binding -
return handling

Iteration is usually faster and safer in Python for large depths.

But recursion is fine for: - small depths - tree traversal when depth is
bounded and readability matters

------------------------------------------------------------------------

# 6) Production Patterns

## 6.1 Tree/AST traversal

When depth is limited or controlled.

## 6.2 Divide-and-conquer with bounded recursion

E.g., quicksort-like algorithms (but Python's `sort` is built-in and
better).

## 6.3 Prefer explicit stack for unbounded depth

Convert recursion to iteration with your own stack when depth may be
large.

------------------------------------------------------------------------

# 7) Pitfalls

-   missing base case → infinite recursion
-   insufficient progress → recursion depth exceeded
-   uncontrolled recursion under user input → DoS risk in web services

------------------------------------------------------------------------

# 8) Interview Traps

1)  Python does not do tail-call optimization
2)  Recursion limit prevents C stack overflow, not just "Python safety"
3)  Raising recursion limit can crash the interpreter
4)  Iteration is usually preferred for deep workflows

------------------------------------------------------------------------

# 9) Summary

Recursion is a correctness tool for naturally recursive structures, but
in backend systems you must control depth and prefer iteration where
depth can grow under load or user input.

------------------------------------------------------------------------
