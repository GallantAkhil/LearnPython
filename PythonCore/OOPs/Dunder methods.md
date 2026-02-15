# PythonCore → OOP Deep Dive → Dunder Methods (Magic Methods)

# 1) Dunder Methods Are Protocol Hooks

Many Python language features translate into dunder calls.

Examples: - `a + b` → `a.__add__(b)` (with fallback to
`b.__radd__(a)`) - `len(x)` → `x.__len__()` (via slot) - `for x in it` →
`it.__iter__()` + iterator `__next__()` - `with obj` → `__enter__` /
`__exit__` - `obj.attr` → `__getattribute__`, descriptors, etc.

------------------------------------------------------------------------

# 2) Slot Dispatch vs Attribute Lookup (CPython)

Important CPython detail:

Many operations do NOT do normal attribute lookup each time. Instead,
CPython uses **type slots** (C-level function pointers) for performance.

Example: - `len(x)` calls the `sq_length` slot if present - numeric ops
call `nb_add`, `nb_multiply`, etc.

So simply adding a `__len__` method at runtime to an instance won't
affect `len(x)`; it must be on the type for the slot to be set properly.

Engineering implication: \> Dunder methods must usually be defined on
the class to participate in the protocol efficiently/correctly.

------------------------------------------------------------------------

# 3) Core Dunder Categories (Practical Set)

## 3.1 Construction & representation

-   `__new__`, `__init__`
-   `__repr__` (debug), `__str__` (user)
-   `__format__`

## 3.2 Attribute access

-   `__getattribute__` (intercepts everything; dangerous)
-   `__getattr__` (fallback)
-   `__setattr__`, `__delattr__`
-   descriptors: `__get__`, `__set__`, `__delete__`

## 3.3 Comparison and hashing

-   `__eq__`, `__lt__`, etc.
-   `__hash__` (must align with equality)

Rule: If `a == b`, then `hash(a) == hash(b)` must hold (for hashables).

## 3.4 Numeric operations

-   `__add__`, `__radd__`, `__iadd__`, etc.
-   beware: `__iadd__` may mutate or return new object

## 3.5 Container protocol

-   `__len__`, `__iter__`, `__next__`
-   `__getitem__`, `__setitem__`, `__contains__`

## 3.6 Context management

-   `__enter__`, `__exit__`

## 3.7 Callables

-   `__call__`

------------------------------------------------------------------------

# 4) Correctness Pitfalls

## 4.1 Equality/Hash invariants

If you implement `__eq__` on a mutable object and keep it hashable, you
can break dict/set behavior. Often you should set `__hash__ = None` to
make it unhashable if equality is custom and object is mutable.

## 4.2 Comparison total ordering

If you implement only `__lt__`, you may get inconsistent sorting unless
you provide consistent methods or use `functools.total_ordering` (with
performance trade-offs).

## 4.3 `__repr__` should be unambiguous

In production debugging, `__repr__` matters more than `__str__`.

------------------------------------------------------------------------

# 5) Performance Notes

-   Slot dispatch is fast when methods are defined on the class
-   Implementing "smart" dynamic behavior via `__getattribute__` can
    destroy performance
-   Prefer explicit methods + properties over heavy magic

------------------------------------------------------------------------

# 6) Engineering Patterns

-   Use dunders to integrate domain objects naturally (e.g., money,
    durations, identifiers)
-   Implement `__enter__/__exit__` for resource safety (DB sessions,
    locks)
-   Implement iteration protocols for streaming objects
-   Use `__slots__` + dunders for memory-efficient DTOs

------------------------------------------------------------------------

# 7) Interview Traps

1)  Many builtins use slot dispatch; dunders must be on the type
2)  `__repr__` vs `__str__` difference
3)  hash/eq invariant
4)  `__iadd__` may mutate or allocate
5)  `__getattribute__` intercepts everything

------------------------------------------------------------------------

**End of document.**
