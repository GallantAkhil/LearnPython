# PythonCore → OOP Deep Dive → Encapsulation in Python

# 1) Encapsulation: What It Means Here

Encapsulation is about: - hiding implementation details - enforcing
invariants - controlling mutation and side effects - providing stable
interfaces

Python provides mechanisms, but relies heavily on convention.

------------------------------------------------------------------------

# 2) Conventions: `_name` and `__name`

## 2.1 Single underscore `_x`

Means "internal/private by convention". Tooling (linters, IDEs) respects
it.

## 2.2 Double underscore `__x` (name mangling)

Python rewrites `__x` inside a class to `_ClassName__x`.

Purpose: - avoid accidental override collisions in subclasses - not for
security

Example: - `self.__token` becomes `self._MyClass__token`

You can still access it if you know the mangled name.

------------------------------------------------------------------------

# 3) Properties as Encapsulation Tools

Use `@property` to expose computed or validated access while keeping
storage private:

``` python
class User:
    def __init__(self):
        self._age = 0

    @property
    def age(self):
        return self._age

    @age.setter
    def age(self, v):
        if v < 0:
            raise ValueError
        self._age = v
```

This enforces invariants at assignment time.

Performance note: - properties add call overhead per access; avoid in
ultra-hot loops.

------------------------------------------------------------------------

# 4) Descriptors (Advanced Encapsulation)

Properties are descriptors. You can build custom descriptors for: -
validation - lazy loading - type enforcement - caching

But heavy descriptor magic can reduce readability and complicate
debugging.

------------------------------------------------------------------------

# 5) Encapsulation at Module/Package Level

Python often uses modules to encapsulate: - underscore-prefixed
"private" functions - explicit `__all__` exports - internal packages not
exposed as API

This is a stronger encapsulation boundary than class-private
conventions.

------------------------------------------------------------------------

# 6) Production Patterns

-   Keep state internal; expose methods not raw attributes when
    invariants matter
-   Prefer immutable data structures for shared state
-   Use dataclasses/Pydantic models with validation at boundaries
-   Avoid public fields that must later become validated properties
    (breaking change risk)

------------------------------------------------------------------------

# 7) Interview Traps

1)  Python "private" is convention; not enforced
2)  `__name` triggers name mangling, mainly for collision avoidance
3)  `property` is a descriptor; it overrides instance dict lookup
4)  Encapsulation is best done via stable interfaces, not access
    restriction tricks

------------------------------------------------------------------------

