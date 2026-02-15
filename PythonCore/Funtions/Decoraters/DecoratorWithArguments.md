# PythonCore → Functions (Advanced) → Decorators With Arguments

# 1) Conceptual Model

A decorator with arguments is a **decorator factory**.

``` python
@deco(x=1, y=2)
def f(...):
    ...
```

Expands to:

``` python
f = deco(x=1, y=2)(f)
```

So you have 3 layers:

1)  Factory: `deco(x=1,y=2)` → returns decorator
2)  Decorator: takes function `f` → returns wrapper
3)  Wrapper: runs on each call of the decorated function

------------------------------------------------------------------------

# 2) Correct Template

``` python
from functools import wraps

def deco(*, retries=3):
    def decorator(fn):
        @wraps(fn)
        def wrapper(*args, **kwargs):
            # use retries from outer scope
            ...
            return fn(*args, **kwargs)
        return wrapper
    return decorator
```

------------------------------------------------------------------------

# 3) Execution Timing (Critical for Production)

-   Factory executes at definition time (import/class definition time)
-   Decorator executes immediately after function object is created
-   Wrapper executes at call time

Implication: - expensive setup in factory happens once (good) - dynamic
config must not be captured incorrectly (pitfall)

------------------------------------------------------------------------

# 4) Pitfalls

## 4.1 Shared mutable config

If you capture a mutable object (dict/list) in the factory, it is shared
across all calls and possibly across functions if reused.

Use immutables or copy defensively.

## 4.2 Async functions need async wrappers

If `fn` is async, wrapper must be `async def` and `await fn(...)`.

Otherwise you return a coroutine and break calling semantics.

## 4.3 Signature loss if you skip wraps

Frameworks break (FastAPI, CLI parsers, etc.). Always use `@wraps`
unless you intentionally change the public signature.

## 4.4 Retrying decorators and exception semantics

Retries can hide bugs, amplify load, and break idempotency unless
carefully designed (especially for DB writes and HTTP POST).

------------------------------------------------------------------------

# 5) Production Patterns

## 5.1 Retry with budget

-   max retries
-   exponential backoff
-   jitter
-   only retry on transient exceptions
-   preserve cancellation semantics

## 5.2 Caching decorator with sentinel

Must distinguish: - cached "None" result - cache miss

Use sentinel for cache miss markers.

## 5.3 Auth decorator

Prefer framework-native middleware/dependencies where possible to avoid
signature/typing complexity.

------------------------------------------------------------------------

# 6) Interview Traps

1)  `@deco(x)` is `f = deco(x)(f)`
2)  Factory runs at import/definition time, not call time
3)  Parameterized decorators rely on closures
4)  Async requires async wrapper
5)  Shared state through captured mutables is a common bug

------------------------------------------------------------------------

# 7) Summary

Decorators with arguments are a 3-stage pipeline. Correctness depends on
understanding evaluation time and closure semantics, and on preserving
signatures for tooling/frameworks.

------------------------------------------------------------------------
