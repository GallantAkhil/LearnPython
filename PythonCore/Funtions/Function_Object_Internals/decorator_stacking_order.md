# PythonCore → Functions (Advanced) → Decorator Stacking Order

# 1) The Rule (Memorize)

Given:

``` python
@A
@B
@C
def f(...):
    ...
```

This expands to:

``` python
f = A(B(C(f)))
```

So: - **C** decorates `f` first - then **B** decorates the result - then
**A** decorates that

Definition-time application: bottom-up.

Call-time nesting: top-down in the wrapper chain.

------------------------------------------------------------------------

# 2) Runtime Call Stack

When you call `f()` after decoration, the outermost wrapper runs first
(A's wrapper), which calls B's wrapper, which calls C's wrapper, which
calls original.

So the runtime stack is: A → B → C → original

This matters for: - ordering of logging/metrics - auth checks and error
raising - caching boundaries - transaction boundaries

------------------------------------------------------------------------

# 3) Introspection and `__wrapped__` Chains

If each decorator uses `functools.wraps`, you get a `__wrapped__` chain:

``` python
f.__wrapped__           # points to inner wrapped function (after A)
f.__wrapped__.__wrapped__  # after B, etc.
```

Frameworks may follow `__wrapped__` to reconstruct signature.

If one decorator fails to use `wraps`, the chain breaks.

------------------------------------------------------------------------

# 4) Ordering Pitfalls (Production)

## 4.1 Caching vs auth

If caching wraps auth incorrectly, you can cache unauthorized responses
or bypass checks.

Typical safe ordering: - auth first (outermost) - caching inside auth
(so only authorized results are cached)

But depends on semantics and multi-tenant caches.

## 4.2 Transaction decorators

Transaction boundaries should typically be outermost around business
logic, but inside auth if auth can raise without DB effects.

## 4.3 Logging and metrics

Outer logging sees the entire duration including inner decorators; inner
logging sees only inner work. Decide intentionally.

------------------------------------------------------------------------

# 5) Debugging Stacked Decorators

Symptoms: - missing annotations/signature - wrong error locations in
tracebacks - unexpected behavior due to decorator order

Approach: 1) inspect `f.__wrapped__` chain 2) log entry/exit in each
wrapper with unique tags 3) ensure each decorator uses `wraps` and
consistent signature forwarding

------------------------------------------------------------------------

# 6) Interview Traps

1)  Stacked decorators apply bottom-up: `A(B(C(f)))`
2)  Runtime wrapper call order is outer-to-inner
3)  One missing `wraps` breaks introspection chain
4)  Order affects correctness (auth/caching/transactions)

------------------------------------------------------------------------

# 7) Summary

Decorator stacking is deterministic and must be designed. Treat
decorator order as part of your public API and your security/performance
model.

