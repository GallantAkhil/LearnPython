# PythonCore → OOP Deep Dive → Instance vs Class Variables

# 1) Conceptual Model

-   **Class variables** live on the class object (`Class.__dict__`)
-   **Instance variables** live on the instance (`obj.__dict__`) unless
    slots

``` python
class A:
    x = 10          # class variable
    def __init__(self):
        self.y = 20 # instance variable
```

------------------------------------------------------------------------

# 2) Lookup & Shadowing Mechanics

Access order for `obj.attr` (simplified):

1)  Data descriptors on class (e.g., property)
2)  `obj.__dict__` (instance attributes)
3)  `Class.__dict__` and bases via MRO (class attributes, methods)
4)  `__getattr__` fallback

Shadowing:

``` python
A.x = 10
a = A()
a.x = 99  # creates instance attribute 'x' (shadows class x)
A.x       # still 10
a.x       # 99
```

Deleting `a.x` reverts to class attribute resolution:

``` python
del a.x
a.x  # 10 (from class)
```

------------------------------------------------------------------------

# 3) Mutability Trap: Shared Class-Level Containers

Bad:

``` python
class Cache:
    store = {}   # shared across all instances!
```

All instances share `store`. This is sometimes intentional, usually a
bug.

Better patterns: - instance attribute in `__init__` - or explicit
singleton/shared cache with clear naming - or use dataclasses with
`default_factory`

Example fix:

``` python
class Cache:
    def __init__(self):
        self.store = {}
```

------------------------------------------------------------------------

# 4) When Class Variables Are Correct

## 4.1 Constants and configuration

``` python
class HTTP:
    TIMEOUT = 5
```

## 4.2 Shared resources by design

-   connection pools
-   compiled regex cache
-   shared metrics registry

But in production: - shared state must be thread-safe - document the
shared semantics explicitly

------------------------------------------------------------------------

# 5) Dataclasses & Defaults

Dataclasses enforce correct default patterns:

``` python
from dataclasses import dataclass, field

@dataclass
class User:
    tags: list[str] = field(default_factory=list)  # per instance
```

If you do `tags: list[str] = []`, you recreate the shared mutable
default trap, but now at class level.

------------------------------------------------------------------------

# 6) Descriptor and Class Variable Interactions

If `x` is a data descriptor on the class (property), it will override
instance dict.

This affects "instance variable vs class variable" reasoning: - some
names are controlled by descriptor objects, not plain values

------------------------------------------------------------------------

# 7) Production Patterns

## 7.1 Readonly config + per-instance state

Use class variables for constants, instance vars for state.

## 7.2 Caching patterns

-   Per-instance cache: `self._cache`
-   Per-class cache: `cls._cache` (shared across instances)
-   Global cache: module-level `_CACHE`

Pick intentionally and enforce locking if needed.

## 7.3 ORM models

Many ORMs store metadata on class (fields, table name) and state on
instance (row values). This is a real-world example of class vs instance
separation.

------------------------------------------------------------------------

# 8) Performance Notes

-   Instance dict lookup involves hashing → relatively expensive
-   Class attribute lookup also hashes but may be cached in CPython's
    attribute cache mechanisms (version-dependent)
-   `__slots__` can speed and reduce memory by avoiding dict
-   Avoid per-call dynamic attribute creation in hot paths

------------------------------------------------------------------------

# 9) Interview Traps

1)  `a.x = ...` creates instance attribute unless `x` is data descriptor
2)  Shared mutable class variables are shared across instances
3)  Shadowing is name-based; deleting instance attr reveals class attr
4)  Dataclass defaults must use `default_factory` for mutables

------------------------------------------------------------------------

**End of document.**
