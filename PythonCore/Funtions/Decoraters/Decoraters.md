# PythonCore → Functions (Advanced) → Decorators (Core Mechanics)

# 1) Conceptual Model

A decorator is a callable that takes a function and returns a
replacement callable.

``` python
@deco
def f(...):
    ...
```

Is equivalent to:

``` python
def f(...):
    ...
f = deco(f)
```

This is evaluated at **definition time** (when the `def` executes).

------------------------------------------------------------------------

# 2) Wrapper Semantics

Common pattern:

``` python
from functools import wraps

def deco(fn):
    @wraps(fn)
    def wrapper(*args, **kwargs):
        # pre
        out = fn(*args, **kwargs)
        # post
        return out
    return wrapper
```

Key issues decorators introduce: - signature loss (unless preserved) -
metadata loss (`__name__`, `__doc__`, `__module__`, `__annotations__`) -
stacking order complexity (covered in dedicated file) - altered
exception boundaries and tracebacks

------------------------------------------------------------------------

# 3) `functools.wraps` (Why It Matters)

`@wraps(fn)` applies `functools.update_wrapper`, copying metadata from
`fn` to `wrapper` and setting:

-   `wrapper.__wrapped__ = fn`

This is critical for: - `inspect.signature` to recover original
signature via `__wrapped__` chain - documentation generators - FastAPI
dependency and route inspection - decorators that must remain
transparent

Without wraps, frameworks may see wrong signatures and break parameter
injection.

------------------------------------------------------------------------

# 4) Decorators on Methods (Descriptor Interaction)

Functions defined in classes are descriptors. Accessing them creates
bound methods.

When decorating methods: - your decorator runs on the function object at
class creation time - the returned wrapper becomes the function stored
on the class - binding happens when accessed via instance

Pitfall: - wrapper must accept `self` appropriately - async wrappers
must be `async def` or return awaitables properly

------------------------------------------------------------------------

# 5) Performance Considerations

Decorators add overhead: - extra Python call layer(s) - potentially
extra allocations / dict merges - added try/except blocks

In hot paths: - be conservative - consider moving logic into the
function body or using C-level hooks

------------------------------------------------------------------------

# 6) Production Patterns

## 6.1 Cross-cutting concerns

-   logging
-   metrics
-   tracing
-   auth checks
-   caching (with careful invalidation semantics)

## 6.2 Avoid hiding side effects

A decorator that changes meaning (e.g., silently swallowing exceptions)
is a maintenance hazard.

## 6.3 Preserve signature for frameworks

In FastAPI and similar, do: - `@wraps` - or use `fastapi.Depends`
patterns rather than wrapping endpoints heavily

------------------------------------------------------------------------

# 7) Interview Traps

1)  `@deco` is `f = deco(f)` at definition time
2)  Without `wraps`, signature and metadata are lost
3)  Methods are descriptors; binding happens after decoration
4)  Decorators add call overhead

------------------------------------------------------------------------

# 8) Summary

Decorators are powerful but dangerous if you ignore: - signature
preservation - wrapper transparency - method binding details - overhead
and debugging complexity

------------------------------------------------------------------------
