# PythonCore → OOP Deep Dive → Diamond Problem

# 1) The Diamond Shape

Classic diamond:

``` python
class A: ...
class B(A): ...
class C(A): ...
class D(B, C): ...
```

`D` inherits `A` through both `B` and `C`.

The risk: - calling `A` methods twice - inconsistent method selection
order - duplicated initialization

------------------------------------------------------------------------

# 2) How Python Solves It: MRO (C3)

Python computes `D.__mro__` such that: - each class appears once - base
ordering constraints are respected

Typical MRO: `D -> B -> C -> A -> object`

Thus, `A` appears once, and cooperative `super()` chains traverse it
once.

------------------------------------------------------------------------

# 3) The Real Fix: Cooperative `super()`

If `B` and `C` both call `super()`, then `A` is called once.

Bad pattern (direct calls):

``` python
class B(A):
    def __init__(self):
        A.__init__(self)

class C(A):
    def __init__(self):
        A.__init__(self)
```

In `D`, both branches may call `A.__init__` → double-init bug.

Correct pattern:

``` python
class A:
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)

class B(A):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)

class C(A):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)

class D(B, C):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
```

Now `A.__init__` runs exactly once along the MRO chain.

------------------------------------------------------------------------

# 4) Production Pitfalls

-   Double initialization of resources (connections, locks)
-   Duplicate registration in global registries
-   Inconsistent state due to repeated base init logic
-   Hard-to-debug bugs because order depends on base ordering

------------------------------------------------------------------------

# 5) Engineering Guidance

-   Use MI primarily for stateless mixins
-   Ensure all classes use cooperative `super()`
-   Standardize init signature conventions
-   Prefer composition if multiple bases carry real state

------------------------------------------------------------------------

# 6) Interview Traps

1)  Diamond problem arises from shared base duplication
2)  Python solves it with C3 MRO and cooperative `super()`
3)  Direct base calls break the chain and can double-call bases

------------------------------------------------------------------------

**End of document.**
