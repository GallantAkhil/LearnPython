# PythonCore → Functions (Advanced) → `*args` and `**kwargs`

# 1) Conceptual Model

-   `*args` collects extra positional arguments into a tuple
-   `**kwargs` collects extra keyword arguments into a dict

Callee side:

``` python
def f(a, *args, **kwargs):
    ...
```

Caller side (unpacking):

``` python
f(1, *more_pos, **more_kw)
```

------------------------------------------------------------------------

# 2) Binding Rules (The Only Order That Works)

In a function definition, the canonical order is:

1)  positional-only (before `/`)
2)  positional-or-keyword
3)  `*args` (optional)
4)  keyword-only (after `*` or after `*args`)
5)  `**kwargs` (optional)

Example:

``` python
def f(a, b, /, c, *args, d, e=5, **kwargs):
    ...
```

------------------------------------------------------------------------

# 3) Packing Semantics (Callee View)

If you call:

``` python
f(1, 2, 3, 4, 5, d=9, x=10)
```

Bindings: - `a=1`, `b=2`, `c=3` - `args=(4,5)` - `d=9`, `e=5` -
`kwargs={'x':10}`

Important: - `args` is always a tuple - `kwargs` is always a dict (newly
created, unless optimized in some paths)

------------------------------------------------------------------------

# 4) Unpacking Semantics (Caller View)

### 4.1 Positional unpacking

``` python
f(*iterable)
```

The iterable is expanded into positional arguments.

### 4.2 Keyword unpacking

``` python
f(**mapping)
```

The mapping is expanded into keyword arguments.

Constraints: - keys must be strings - duplicates after merging raise
`TypeError`

### 4.3 Merge order matters

``` python
f(**a, **b)
```

If keys overlap: - later mapping wins at merge time - but duplicates
against explicit keywords can error depending on combination

Be careful when forwarding kwargs in wrappers.

------------------------------------------------------------------------

# 5) CPython Internals (High-Level)

At call time, CPython constructs: - an argument array for positional
args - a dict for keyword args (or a compact "vectorcall" representation
in newer versions)

In CPython 3.8+, many calls use **vectorcall** (PEP 590) to reduce
overhead by passing arguments in a C array without building temporary
tuples/dicts in some cases.

However, the moment you use: - `*args` packing in callee - or `**kwargs`
packing you often force tuple/dict materialization.

Engineering takeaway: \> Variadic forwarding (`*args, **kwargs`) is
flexible but can be costlier than explicit signatures.

------------------------------------------------------------------------

# 6) Pitfalls

## 6.1 Silent API acceptance

`**kwargs` can hide typos:

``` python
def f(**kwargs):
    ...
f(tiemout=3)  # typo silently accepted unless you validate
```

In backend code, this can become a production outage.

Mitigation: - explicit keyword-only parameters - strict validation of
kwargs keys - use TypedDict / Pydantic models at boundaries

## 6.2 Wrapper functions that change semantics

Decorators that accept `*args, **kwargs` may accept calls that the
wrapped function would reject (pos-only/kw-only constraints).

Use `functools.wraps` + signature preservation when needed.

## 6.3 Performance: repeated dict merges

Forwarding kwargs like:

``` python
f(**defaults, **kwargs)
```

creates merged dicts. In hot paths, avoid repeated merges; precompute or
keep explicit params.

------------------------------------------------------------------------

# 7) Production Patterns

## 7.1 Forward-compatible API design

Accept `**kwargs` only at boundaries where you intentionally permit
extension, and validate keys.

## 7.2 Decorator forwarding

Canonical pattern:

``` python
from functools import wraps

def deco(fn):
    @wraps(fn)
    def wrapper(*args, **kwargs):
        return fn(*args, **kwargs)
    return wrapper
```

But be aware it may not enforce original call constraints unless
signature is preserved.

## 7.3 Request building

`*args/**kwargs` are useful for building flexible adapters around
HTTP/DB clients, but enforce strictness at the edge.

------------------------------------------------------------------------

# 8) Interview Traps

1)  Correct parameter order matters in definitions
2)  `*args` is tuple, `**kwargs` is dict
3)  `**` keys must be strings
4)  Duplicate keyword values raise `TypeError`
5)  Vectorcall exists but variadic packing may force allocations

------------------------------------------------------------------------

# 9) Summary

-   `*args/**kwargs` provide flexibility via packing/unpacking
-   CPython can optimize calls, but variadic usage can force allocations
-   Use explicit keyword-only params for strict APIs; validate kwargs
    when you allow extension
-   Preserve signature if wrappers must be introspection-friendly

------------------------------------------------------------------------
