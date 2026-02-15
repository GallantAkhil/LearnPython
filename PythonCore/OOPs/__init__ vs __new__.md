# PythonCore → OOP Deep Dive → `__init__` vs `__new__`

# 1) Conceptual Model

Construction pipeline for `C(...)`:

1)  `obj = C.__new__(C, *args, **kwargs)`
2)  if `obj` is instance of `C`, then call
    `C.__init__(obj, *args, **kwargs)`
3)  return `obj`

Key points: - `__new__` is a **staticmethod-like** method (receives
`cls`) - `__init__` receives `self` - `__init__` must return `None`

------------------------------------------------------------------------

# 2) Why Two Phases Exist

-   `__new__`: memory allocation + instance creation
-   `__init__`: set attributes, validate invariants

For many classes you only override `__init__`.

You override `__new__` when: - subclassing immutables (`int`, `str`,
`tuple`) - implementing singletons / caching instances - controlling
allocation or returning alternate types - metaclass programming and
proxies

------------------------------------------------------------------------

# 3) Immutable Types: `__new__` Is Mandatory

Example: subclass `str`

``` python
class UserId(str):
    def __new__(cls, value):
        value = str(value).strip()
        return super().__new__(cls, value)
```

`str` instances are immutable; you can't "set" the internal string
content in `__init__`. So you must create the correct value in
`__new__`.

Same for `int`, `tuple`, `frozenset`, `bytes`.

------------------------------------------------------------------------

# 4) `__new__` Can Return Non-Instance (Advanced)

If `__new__` returns an object that is NOT an instance of `cls`,
`__init__` is skipped.

This is used for: - caching / flyweight patterns - factories
masquerading as classes - singleton implementations

But in production, this can confuse tooling and type systems. Prefer
explicit factories unless you need deep control.

------------------------------------------------------------------------

# 5) CPython Details (High Level)

-   `type.__call__` orchestrates instance creation
-   it invokes `__new__`, then `__init__` conditionally
-   builtins may implement `tp_new` and `tp_init` slots in C
-   for Python classes, slots usually point to generic wrappers that
    call your Python-level `__new__/__init__`

------------------------------------------------------------------------

# 6) Dataclasses and `__init__`

Dataclasses generate `__init__` automatically. If you need `__new__`
customization, do it manually, but be cautious: dataclasses and
`__new__` interactions can get complex.

For immutables, consider `frozen=True` and `__post_init__` for
validation (note: still not for changing immutable core value).

------------------------------------------------------------------------

# 7) Performance & Correctness Pitfalls

## 7.1 Heavy work in `__init__`

If object creation is frequent, `__init__` hot path matters. Keep it
cheap; move heavy work to explicit methods.

## 7.2 Forgetting to call `super().__init__`

In inheritance, especially multiple inheritance, you must cooperate with
MRO via `super()`.

## 7.3 Misusing `__new__` for normal mutable classes

Overriding `__new__` when not needed often creates subtle bugs and makes
code harder to reason about.

------------------------------------------------------------------------

# 8) Production Patterns

## 8.1 Validation in `__init__`

Good for enforcing invariants at boundaries.

## 8.2 Controlled construction via classmethods

Often better than overriding `__new__`:

``` python
class User:
    @classmethod
    def from_row(cls, row):
        ...
```

## 8.3 Subclassing immutables

Use `__new__` for normalization and canonicalization.

------------------------------------------------------------------------

# 9) Interview Traps

1)  `__new__` creates instance; `__init__` initializes it
2)  `__init__` must return `None`
3)  If `__new__` returns non-instance, `__init__` is skipped
4)  Immutable subclasses must override `__new__`, not just `__init__`
5)  `type.__call__` coordinates the whole process

------------------------------------------------------------------------

**End of document.**
