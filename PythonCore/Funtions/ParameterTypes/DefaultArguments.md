# PythonCore → Functions (Advanced) → Default Arguments (Evaluation, Storage, Pitfalls)

# 1) Conceptual Model

Defaults are evaluated **once**, at function definition time.

``` python
def f(x, y=10):
    ...
```

`y=10` is computed when the `def` executes (module import/runtime), not
when the function is called.

This is a feature, not a bug.

------------------------------------------------------------------------

# 2) Where Defaults Are Stored

Python function object stores:

-   `f.__defaults__` → tuple of positional defaults (for trailing
    positional-or-keyword parameters)
-   `f.__kwdefaults__` → dict of keyword-only defaults

Example:

``` python
def g(a, b=2, *, c=3): ...
g.__defaults__     # (2,)
g.__kwdefaults__   # {'c': 3}
```

These are part of the function object and persist across calls.

------------------------------------------------------------------------

# 3) CPython Call-Time Application

At call time, CPython: 1) binds provided arguments into local slots 2)
if some trailing parameters are missing, fills from `__defaults__` 3)
fills missing keyword-only params from `__kwdefaults__` 4) errors if
required parameters remain unbound

This happens before executing the function body.

------------------------------------------------------------------------

# 4) The Mutable Default Trap (Root Cause)

Bad:

``` python
def f(x, cache={}):
    cache[x] = True
    return cache
```

Why this breaks: - `{}` is created once - same dict reused for every
call - calls share state unintentionally

Demonstration: - repeated calls mutate the same object

Correct pattern:

``` python
def f(x, cache=None):
    if cache is None:
        cache = {}
    ...
```

This uses `None` as an "argument not provided" sentinel.

------------------------------------------------------------------------

# 5) When `None` Is Not Enough: Real Sentinel Pattern

Sometimes `None` is a valid input distinct from "not provided" (e.g.,
`timeout=None` meaning no timeout).

Use a unique sentinel:

``` python
_MISSING = object()

def request(timeout=_MISSING):
    if timeout is _MISSING:
        timeout = 5
    ...
```

This prevents conflating: - "no argument provided" - "argument provided
as None"

------------------------------------------------------------------------

# 6) Performance & Memory Considerations

-   Defaults avoid re-allocating common objects each call
-   For immutable defaults (ints, tuples, frozensets), this is efficient
-   For mutable defaults, shared state is usually a bug

Sometimes you *want* shared state (e.g., memoization), but be explicit
and document it.

------------------------------------------------------------------------

# 7) Production Patterns

## 7.1 Safe optional containers

``` python
def add_user(user, users=None):
    if users is None:
        users = []
    users.append(user)
    return users
```

## 7.2 Stable default configuration objects

Prefer immutables: - tuples - frozen dataclasses - mappingproxy
patterns - `types.SimpleNamespace` only if you're sure about mutation
semantics

## 7.3 Caching with explicit shared default (rare)

If you intentionally share state: - name it explicitly (`_cache`) - keep
it module-level or closure-contained - add locking if concurrent

------------------------------------------------------------------------

# 8) Interview Traps

1)  Defaults are evaluated at **definition time**, not call time
2)  Mutable defaults persist across calls
3)  `__defaults__` stores positional defaults, `__kwdefaults__` stores
    keyword-only defaults
4)  Use `None` sentinel or a unique sentinel when `None` is meaningful
    input

------------------------------------------------------------------------

# 9) Summary

-   Defaults live on the function object and persist
-   CPython fills missing parameters from these stored defaults
-   Avoid mutable defaults unless you intentionally want shared state
-   Use sentinel objects for robust API semantics

------------------------------------------------------------------------
