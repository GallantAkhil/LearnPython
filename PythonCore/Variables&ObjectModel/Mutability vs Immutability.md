# PythonCore → Variables & Object Model → Mutability vs Immutability

**Systems-Level Deep Dive (object state + hash invariants + thread
safety)**

------------------------------------------------------------------------

## 1) Conceptual Model

Mutable objects: - state can change in place

Immutable objects: - state cannot change after creation

Examples: - mutable: list, dict, set - immutable: int, str, tuple (if
elements immutable)

------------------------------------------------------------------------

## 2) Hash Contract

Immutable objects can be hashable.

Rule: If object is hashable, its hash must not change.

Therefore: - mutable objects are generally unhashable - immutable
objects safe for dict keys

------------------------------------------------------------------------

## 3) Memory Behavior

Mutation modifies object in place. Rebinding creates new object.

``` python
x = [1]
x.append(2)   # same object mutated
```

------------------------------------------------------------------------

## 4) Performance

Mutation avoids allocation. Immutability avoids accidental shared state
bugs.

Trade-offs depend on workload.

------------------------------------------------------------------------

## 5) Engineering Guidance

-   Prefer immutability for shared state
-   Use mutable for accumulation patterns
-   Never use mutable default arguments

------------------------------------------------------------------------
