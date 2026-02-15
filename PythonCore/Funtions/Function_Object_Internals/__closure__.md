# PythonCore → Functions (Advanced) → Function Object Internals: `__closure__`

# 1) What `__closure__` Is

For a function that captures free variables, `fn.__closure__` is: -
`None` if no captured variables - otherwise a tuple of **cell objects**

Each cell holds one captured variable value.

Example:

``` python
def outer(x):
    def inner(y):
        return x + y
    return inner

f = outer(10)
f.__closure__          # (cell,)
f.__closure__[0].cell_contents  # 10
```

------------------------------------------------------------------------

# 2) Relationship to `__code__`

Captured variables are described by `fn.__code__.co_freevars`.

These names correspond to the cells in `__closure__` by position.

------------------------------------------------------------------------

# 3) Memory Lifetime Implications

Captured objects remain alive as long as the closure exists:

-   if you capture a large object, it won't be GC'd until the closure is
    released
-   this can cause memory retention bugs in long-lived services

Example risk: - capturing DB connections, huge caches, request objects,
etc.

Best practice: - capture only what you need - avoid capturing
request-scoped objects into global closures

------------------------------------------------------------------------

# 4) Debugging Patterns

If you see unexpected memory growth or state retention: - inspect
`fn.__closure__` - print `co_freevars` - look at `cell_contents` types
and sizes

This is extremely useful for diagnosing leaked references in decorators
and factories.

------------------------------------------------------------------------

# 5) Production Guidance

Closures are great for: - factories - DI - encapsulated state

But in backend services: - ensure captured state is intentional - avoid
retaining per-request objects - consider explicit classes when state
needs lifecycle management

------------------------------------------------------------------------

# 6) Interview Traps

1)  `__closure__` holds cells, not direct values
2)  Cells correspond to `co_freevars`
3)  Closures can keep objects alive unexpectedly

------------------------------------------------------------------------
