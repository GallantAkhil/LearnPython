# PythonCore → Variables & Object Model → Identity vs Equality (`is` vs `==`)

**Systems-Level Deep Dive (pointer identity + rich comparison protocol +
performance)**

------------------------------------------------------------------------

## 1) Identity (`is`)

Checks pointer equality.

``` python
a is b
```

True only if both names reference same object.

Implemented as pointer comparison.

------------------------------------------------------------------------

## 2) Equality (`==`)

Calls `__eq__` method via rich comparison protocol.

``` python
a == b
```

Dispatches to type's comparison slot.

------------------------------------------------------------------------

## 3) CPython Details

-   `is` → direct pointer compare (O(1))
-   `==` → method dispatch, may be O(n)

For containers, equality is structural and recursive.

------------------------------------------------------------------------

## 4) Special Case: None

Always use:

``` python
x is None
```

Never `== None`.

Reason: - None is singleton - identity is correct semantic

------------------------------------------------------------------------

## 5) Performance Notes

In tight loops: - `is` is faster than `==` - But correctness \>
micro-optimization

------------------------------------------------------------------------

## 6) Pitfalls

-   Small ints and strings may appear identical due to interning
-   Do not rely on `is` for value comparison

------------------------------------------------------------------------
