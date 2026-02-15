# PythonCore → Functions (Advanced) → Function Object Internals: `__defaults__` and `__kwdefaults__`

# 1) What These Attributes Are

For a function `f`:

-   `f.__defaults__`: tuple of defaults for trailing
    positional-or-keyword params
-   `f.__kwdefaults__`: dict of defaults for keyword-only params

Example:

``` python
def f(a, b=2, *, c=3): ...
f.__defaults__     # (2,)
f.__kwdefaults__   # {'c': 3}
```

These are stored on the function object and persist for the lifetime of
the function.

------------------------------------------------------------------------

# 2) CPython Call Binding Uses These

During call binding: - missing positional-or-keyword trailing args →
filled from `__defaults__` (aligned to the end of parameter list) -
missing keyword-only args → filled from `__kwdefaults__` by name

If required args remain missing → `TypeError`.

------------------------------------------------------------------------

# 3) Mutation Hazards

If a default object is mutable and mutated, all future calls observe it.

You can even mutate it through `__defaults__` references in reflective
code.

This is why mutable defaults are dangerous unless intentional.

------------------------------------------------------------------------

# 4) Debugging Patterns

If a function behaves "statefully" unexpectedly, check:

-   `f.__defaults__`
-   `id(f.__defaults__[...])` across calls
-   whether the default object is being mutated

------------------------------------------------------------------------

# 5) Engineering Patterns

-   Prefer immutable defaults (numbers, strings, tuples, frozensets)
-   Use `None` or unique sentinels for optional containers
-   For tri-state semantics where `None` is meaningful, use a sentinel
    object

------------------------------------------------------------------------

# 6) Interview Traps

1)  Defaults are stored on the function object
2)  `__defaults__` is tuple, `__kwdefaults__` is dict
3)  Mutable defaults persist across calls and can be mutated

------------------------------------------------------------------------
