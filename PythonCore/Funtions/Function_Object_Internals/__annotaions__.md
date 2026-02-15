# PythonCore → Functions (Advanced) → Function Object Internals: `__annotations__`

# 1) Conceptual Model

Annotations attach metadata to parameters and return values:

``` python
def f(x: int, y: str) -> bool:
    ...
```

They do not enforce types by themselves. They are stored as a dictionary
on the function object:

``` python
f.__annotations__
```

------------------------------------------------------------------------

# 2) Evaluation Rules

Historically, annotations were evaluated at function definition time.

Modern Python supports **postponed evaluation** of annotations (strings)
to avoid runtime import cycles and heavy typing costs.

Common pattern:

``` python
from __future__ import annotations
```

This stores annotations as strings, deferring evaluation.

To resolve to real types, use:

``` python
from typing import get_type_hints
get_type_hints(f)
```

This can evaluate strings, handle forward refs, and apply
globalns/localns context.

------------------------------------------------------------------------

# 3) CPython Storage

-   `__annotations__` is a dict
-   keys are parameter names and `'return'`
-   values are whatever expressions produced (types, strings, typing
    constructs)

Example:

``` python
{'x': <class 'int'>, 'return': <class 'bool'>}
```

or with postponed annotations:

``` python
{'x': 'int', 'return': 'bool'}
```

------------------------------------------------------------------------

# 4) Framework Implications (FastAPI / Pydantic)

FastAPI uses annotations to: - build request models - parse/validate
inputs - generate OpenAPI schemas - perform dependency injection

Decorator/wrapper pitfalls: - losing annotations breaks validation -
wrapping endpoints without `functools.wraps` can break DI and docs

Production rule: \> Preserve `__annotations__` and `__wrapped__` chain
for introspection-based frameworks.

------------------------------------------------------------------------

# 5) Pitfalls

## 5.1 Circular imports

Type hints referencing classes across modules can create import cycles.
Use: - postponed annotations - `typing.TYPE_CHECKING` blocks - local
imports inside type-checking sections

## 5.2 Runtime overhead

`typing` constructs can be expensive to evaluate. Postpone where
appropriate and resolve only when needed.

## 5.3 `get_type_hints` needs namespaces

If you resolve hints in dynamic contexts, pass `globalns` and `localns`
explicitly.

------------------------------------------------------------------------

# 6) Production Patterns

-   Use `from __future__ import annotations` in library code to reduce
    import-time costs
-   Use `get_type_hints` for runtime resolution in frameworks/tools
-   Keep annotations stable and avoid heavy runtime side effects
-   Use Protocols and TypedDict for clean boundaries

------------------------------------------------------------------------

# 7) Interview Traps

1)  Annotations are stored in `__annotations__` and aren't enforced
2)  With postponed evaluation, annotations may be strings
3)  Frameworks rely on annotations; losing them breaks behavior
4)  `get_type_hints` resolves forward refs but may import/evaluate types

------------------------------------------------------------------------
