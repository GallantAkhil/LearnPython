# PythonCore → Functions (Advanced) → Positional-only Parameters (`/`)

# 1) Conceptual Model

A **positional-only** parameter can be supplied **only by position**,
not by keyword.

Syntax uses `/` in the signature:

``` python
def f(a, b, /, c, d):
    ...
```

Rules: - `a`, `b` are positional-only - `c`, `d` can be passed
positionally or by keyword

Examples:

``` python
f(1, 2, 3, 4)           # OK
f(1, 2, c=3, d=4)       # OK
f(a=1, b=2, c=3, d=4)   # TypeError (a,b positional-only)
```

This feature formalized and exposed in Python 3.8+ (PEP 570).

------------------------------------------------------------------------

# 2) Why Positional-only Exists (API Design)

## 2.1 Backward-compatible API evolution

If you allow keywords today, you may be **locked into parameter names
forever** because callers can do:

``` python
f(limit=10)
```

Renaming `limit` would break callers.

With positional-only, you can later rename internal parameter names
without breaking public API, because users cannot depend on the name.

## 2.2 Builtins already had it (now Python can express it)

Many C-implemented builtins accept positional-only arguments
historically. `/` makes that expressible in Python code and visible to
introspection.

## 2.3 Performance and calling conventions (practical)

Keyword argument handling: - requires mapping keywords to parameters -
requires duplicate checks (positional + keyword) - involves dict/tuple
handling internally

Positional-only can simplify parsing and reduce error surface. In
Python-level code the win is usually small; but in CPython internals and
API stability it's big.

------------------------------------------------------------------------

# 3) Binding Semantics (Exact Rules)

Given:

``` python
def f(a, b, /, c=3, *, d, e=5):
    ...
```

Parameter kinds: - `a, b`: positional-only - `c`: positional-or-keyword
(default) - `d`: keyword-only (required) - `e`: keyword-only (default)

Allowed calls:

``` python
f(1, 2, d=9)                 # c default
f(1, 2, 10, d=9, e=11)        # c positional
f(1, 2, c=10, d=9)            # c by keyword
```

Illegal calls:

``` python
f(a=1, b=2, d=9)              # a,b are positional-only
f(1, 2, 3, 4, d=9)             # too many positionals
f(1, 2)                        # missing keyword-only d
```

------------------------------------------------------------------------

# 4) CPython Internals: How Calls Are Parsed

## 4.1 Function signature is encoded in the code object

A Python function has a `__code__` object. Its metadata includes counts:

-   `co_argcount` → number of positional-or-keyword parameters (includes
    positional-only? see below)
-   `co_posonlyargcount` → number of positional-only parameters
-   `co_kwonlyargcount` → number of keyword-only parameters
-   `co_varnames` → local variable names including arguments

For modern CPython: - positional-only count is tracked explicitly as
`co_posonlyargcount`

## 4.2 Call pipeline (high-level)

When you call `f(...)`, CPython must: 1) allocate a frame (or reuse fast
frame structures in newer versions) 2) map incoming args/kwargs into
"fast locals" array positions 3) apply defaults, `*args`, `**kwargs` 4)
error-check duplicates/missing/unexpected keywords

Positional-only affects step (2): keywords are not allowed to bind those
slots.

## 4.3 Error checks specific to pos-only

When keywords are present, CPython checks if any keyword matches a
pos-only name. If so, it raises `TypeError` with a message like: - "got
some positional-only arguments passed as keyword arguments"

This is a semantic guard to protect API stability.

------------------------------------------------------------------------

# 5) Interaction with `*args` / `**kwargs`

Positional-only works naturally with varargs:

``` python
def f(a, /, *args, **kwargs):
    ...
```

-   `a` must be positional
-   remaining positional args go into `args`
-   keywords go into `kwargs`

But you still cannot pass `a=` as keyword even though `**kwargs` exists;
`a` is not collected into `kwargs`, it is forbidden.

------------------------------------------------------------------------

# 6) Introspection (`inspect.signature`) and Tooling

`inspect.signature(f)` shows `/` and `*` markers.

Why this matters in production: - auto-generated docs (FastAPI, OpenAPI)
may reflect these markers - wrappers/decorators must preserve signature
if you want correct introspection

Use `functools.wraps` and sometimes `inspect.Signature.replace` when
building frameworks.

------------------------------------------------------------------------

# 7) Practical Engineering Patterns

## 7.1 Public API stability

When you publish a library, consider making "implementation detail"
params positional-only so you can rename freely later.

## 7.2 "Sentinel + positional-only" pattern

Useful when you want a private param without keyword exposure:

``` python
_MISSING = object()

def parse(value, /, *, default=_MISSING):
    ...
```

-   `value` must be positional (stable)
-   default is keyword-only (clarity)

## 7.3 Wrapper design

When writing decorators, avoid losing pos-only constraints. If you do
`*args, **kwargs` without forwarding constraints, you can accept calls
the original would reject.

For high correctness wrappers, preserve signature (advanced).

------------------------------------------------------------------------

# 8) Interview Traps

1)  `/` means "everything before is positional-only"
2)  Positional-only is mainly for API stability, not only performance
3)  Keywords cannot bind pos-only even if `**kwargs` is present
4)  `inspect.signature` exposes pos-only and can affect frameworks

------------------------------------------------------------------------

# 9) Summary

-   Positional-only parameters exist for API stability and explicit
    calling conventions
-   CPython tracks them in `co_posonlyargcount`
-   Argument binding forbids keywords for those slots
-   Use in public APIs to prevent name-coupling and allow future
    refactors

------------------------------------------------------------------------
