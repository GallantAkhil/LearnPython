# PythonCore → OOP Deep Dive → MRO & Multiple Inheritance

# 1) Conceptual Model

When a class inherits from multiple bases:

``` python
class C(A, B): ...
```

Python must decide: - where to look for methods/attributes first - how
to combine base class method chains safely

This order is MRO: Method Resolution Order.

------------------------------------------------------------------------

# 2) Attribute Lookup Uses MRO

For `obj.method`, Python searches: - `obj.__dict__` (after data
descriptors) - then `C.__dict__` - then bases in `C.__mro__` order

`C.__mro__` is a tuple starting with `C`, ending with `object`.

------------------------------------------------------------------------

# 3) C3 Linearization (The Algorithm)

Python uses C3 linearization to compute MRO. It guarantees: - local
precedence order (left-to-right bases preserved) - monotonicity
(subclass doesn't break parent ordering constraints) - no duplication

You can introspect:

``` python
C.mro()
C.__mro__
```

If no consistent linearization exists, Python raises `TypeError` at
class creation time.

------------------------------------------------------------------------

# 4) `super()` Semantics (Often Misunderstood)

`super()` does NOT mean "call parent class method." It means:

> "call the next implementation of this method in the MRO after the
> current class."

So in multiple inheritance, `super()` is essential for cooperative
behavior.

Correct cooperative init:

``` python
class A:
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)

class B:
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)

class C(A, B):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
```

This ensures every class in MRO gets a chance to initialize.

------------------------------------------------------------------------

# 5) Common Pitfalls

## 5.1 Direct base calls break cooperativity

``` python
A.__init__(self)  # breaks MRO chain
```

## 5.2 Signature mismatch kills cooperative init

If classes don't accept `*args/**kwargs`, you get `TypeError`.

Design rule: - In MI hierarchies, prefer flexible signatures or shared
parameter conventions.

## 5.3 Diamond inheritance complexity

Covered in dedicated diamond file, but MRO is the solution mechanism.

------------------------------------------------------------------------

# 6) Production Guidance

-   Use multiple inheritance sparingly (mixins)
-   Ensure mixins are small and focused
-   Always use `super()` in mixins, not direct base calls
-   Prefer composition when behavior is not truly "is-a" or needs
    stateful lifecycle

------------------------------------------------------------------------

# 7) Interview Traps

1)  `super()` means "next in MRO", not "direct parent"
2)  MRO is computed via C3 linearization
3)  Attribute lookup uses `__mro__`
4)  Cooperative init requires `super()` across all classes

------------------------------------------------------------------------
