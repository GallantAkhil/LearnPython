# PythonCore → Datatypes → Boolean Logic (`bool`)

# 1) Conceptual Model

### 1.1 `bool` is a real type, but also an `int` subtype

Key invariant:

-   `bool` has exactly **two** values: `True` and `False`
-   In Python's type hierarchy: `bool` **subclasses** `int`

Implications:

``` python
isinstance(True, int)  # True
True + True            # 2
False * 10             # 0
hash(True) == hash(1)  # True
```

This is deliberate: it enables legacy numeric behavior and allows `bool`
to integrate with arithmetic, indexing, bit operations, etc.

### 1.2 Truthiness is a protocol, not a property of the type name

"Truthy/falsy" isn't only about `bool` values. It is determined by the
**truth value testing protocol**:

1)  If object has `__bool__`, call it → must return `True/False` (a
    bool)
2)  Else if object has `__len__`, call it → zero means falsy, nonzero
    truthy
3)  Else default → truthy

`None` is falsy by design (it has a special truth behavior in C).

------------------------------------------------------------------------

# 2) CPython Internal Implementation

This section is CPython-specific. Other interpreters can differ, but the
*language* behavior is the same.

## 2.1 Object representation: `PyBoolObject`

In CPython, `bool` is implemented as a specialized integer object.

Internally (conceptually):

-   There is a `PyBoolObject` struct that behaves like a `PyLongObject`
    with value 0 or 1

-   There are exactly **two singleton instances**:

    -   `_Py_TrueStruct`
    -   `_Py_FalseStruct`

And their addresses are exposed as:

-   `Py_True`
-   `Py_False`

### 2.2 Singletons guarantee

This is why identity checks are valid:

``` python
(True is True)    # always True
(False is False)  # always True
```

Also:

``` python
bool(1) is True
bool(0) is False
```

Because `bool(x)` returns the singleton, not a new object.

### 2.3 Memory behavior

-   `True` and `False` are **immortal singletons** for program lifetime
-   No per-call allocation for truth values
-   Passing booleans around is pointer passing (cheap)

But: - Any *derived* results like `a < b` still create a `PyBoolObject`
reference (returned object is `Py_True`/`Py_False`)

------------------------------------------------------------------------

# 3) Runtime Truthiness Mechanics

## 3.1 `PyObject_IsTrue` (the core truth evaluation)

Any time Python needs truth value (e.g., `if x`, `while x`, `and/or`
short-circuit), CPython effectively does:

-   Fast-path checks for `True`, `False`, `None`
-   Otherwise calls `tp_as_number->nb_bool` (i.e., `__bool__`)
-   Else calls `sq_length` (i.e., `__len__`)
-   Else defaults to truthy

### 3.2 `__bool__` must return `bool`

Important: `__bool__` is required to return a bool, not any truthy
object.

Bad:

``` python
class X:
    def __bool__(self):
        return 1      # WRONG: must return bool
```

This raises `TypeError` in CPython.

### 3.3 `__len__` fallback is powerful and dangerous

If you define `__len__`, you can control truthiness indirectly:

``` python
class Bag:
    def __len__(self):
        return 0

bool(Bag())  # False
```

Pitfall: if `__len__` is expensive (e.g., counts DB rows), `if obj:`
becomes expensive.

------------------------------------------------------------------------

# 4) Boolean Operators (`not`, `and`, `or`) Semantics

## 4.1 `not x`

-   Always returns a **bool**
-   Uses truthiness protocol on `x`, then negates

``` python
not []     # True
not [1]    # False
```

## 4.2 `x and y` / `x or y` do NOT necessarily return bool

This is a core Python semantic and a frequent interview trap:

-   `x and y` returns **x if x is falsy**, else returns y
-   `x or y` returns **x if x is truthy**, else returns y

Examples:

``` python
0 and 999       # 0
123 and 999     # 999
"" or "guest"   # "guest"
"user" or "x"   # "user"
```

This design enables common backend patterns like defaults, lazy
evaluation, and guard expressions.

### 4.3 Short-circuit evaluation guarantees

-   `and` stops evaluating once the result is determined by a falsy left
    operand
-   `or` stops evaluating once result is determined by a truthy left
    operand

``` python
def boom():
    raise RuntimeError("shouldn't run")

False and boom()  # boom() not called
True  or boom()   # boom() not called
```

------------------------------------------------------------------------

# 5) CPython Bytecode & Short-Circuiting

Short-circuiting is not "magic"; it's compiled into jumps.

## 5.1 `if x:` compilation

The interpreter evaluates `x`, then performs a conditional jump based on
truthiness.

Conceptually:

-   Evaluate expression
-   Convert to truth value
-   If false: jump to else / end

## 5.2 `and/or` bytecode patterns (high-level)

For `a and b`:

-   Evaluate `a`
-   If falsy → return `a`
-   Else pop it, evaluate `b`, return `b`

For `a or b`:

-   Evaluate `a`
-   If truthy → return `a`
-   Else pop it, evaluate `b`, return `b`

The key: CPython uses dedicated conditional jump opcodes to avoid
evaluating the RHS unnecessarily.

## 5.3 Practical implication: side effects

Because RHS may never execute, side effects on RHS may not happen:

``` python
flag and do_side_effect()
```

If `flag` is falsy, `do_side_effect()` is skipped entirely.

------------------------------------------------------------------------

# 6) Edge Cases & Pitfalls (Production Reality)

## 6.1 `None` vs `False` --- don't mix semantic meanings

-   `None` often means "unknown / missing / not provided"
-   `False` means "explicit negation"

Bad:

``` python
if not value:
    ...
```

This treats `None`, `0`, `""`, `[]`, `{}`, `False` identically.

Prefer explicit checks when meaning matters:

``` python
if value is None:
    ...
```

## 6.2 NaN truthiness

`float('nan')` is truthy because it is a non-zero float value object.

The real danger is comparison behavior:

``` python
x = float('nan')
x == x  # False
```

So "is this value set?" checks using equality may break.

## 6.3 Containers: emptiness vs existence

-   Empty containers are falsy by `__len__ == 0`
-   But a container might be expensive to compute length for if it's
    lazy/virtual

Use: - `if items:` for in-memory lists/dicts/tuples/sets (cheap) - Avoid
for database-backed lazy collections unless you know cost

## 6.4 numpy / pandas: ambiguous truth value

In numpy, arrays do **not** define a single truthiness value if they
have multiple elements:

``` python
import numpy as np
a = np.array([1, 2, 3])
bool(a)  # ValueError: ambiguous truth value
```

Use explicit reductions:

-   `a.any()`
-   `a.all()`

Similar story with pandas Series/DataFrame.

## 6.5 `and/or` used as ternary (historical anti-pattern)

Old style:

``` python
result = cond and a or b
```

This fails when `a` is falsy:

``` python
cond = True
a = ""
b = "fallback"
(cond and a or b)  # returns "fallback" (WRONG if you wanted "")
```

Use proper ternary:

``` python
result = a if cond else b
```

## 6.6 `is` vs `==`

-   Use `is` for singleton identity (`None`, `True`, `False`)
-   Use `==` for value comparisons

Examples:

``` python
x is None          # correct
x == None          # discouraged (can be overloaded)

flag is True       # usually unnecessary
flag == True       # worse
```

Prefer:

``` python
if flag:
    ...
```

But if tri-state (True/False/None), be explicit.

------------------------------------------------------------------------

# 7) Performance Considerations

## 7.1 Booleans are cheap, but truthiness checks might not be

-   Checking `True/False/None` is fast
-   Calling `__bool__` is a method dispatch (attribute lookup + call)
-   Calling `__len__` might be O(n) for some custom structures

Rule: \> `if obj:` is only "cheap" if `obj.__bool__` / `__len__` is
cheap.

## 7.2 Branching costs are often dominated by Python overhead

In tight loops, the bottleneck is usually: - Python bytecode dispatch -
object allocations - attribute lookups

Not CPU branch misprediction like in C++.

Optimization patterns: - Move invariant checks outside loops - Replace
repeated truthiness calls with a cached boolean - Use local variables
for attribute lookups

## 7.3 Boolean arithmetic can be a micro-optimization tool

Because `bool` is `int`:

``` python
count += (x > threshold)
```

This is common in high-performance Python when avoiding branches, but
only helps if it reduces Python-level overhead (often marginal).

------------------------------------------------------------------------

# 8) Interview-Level Traps

1)  **`bool` is a subclass of `int`**\
    `True + True == 2`

2)  **`and/or` return operands** (not necessarily bool)

3)  **Short-circuit evaluation** prevents RHS execution

4)  **`if not x` conflates many values** (`0`, `""`, `[]`, `None`)

5)  **`__bool__` must return bool** (not `1`, not custom object)

6)  **numpy/pandas ambiguous truth** requires `.any()` / `.all()`

7)  **`is` is identity, `==` is equality**

------------------------------------------------------------------------

# 9) Practical Engineering Patterns (Backend Grade)

## 9.1 Tri-state flags (True/False/None)

Use explicit checks:

``` python
if flag is True:
    ...
elif flag is False:
    ...
else:
    ...  # None / missing
```

This prevents silent mixing of "missing" and "false."

## 9.2 Defaulting with `or` --- only when falsy means "missing"

Good:

``` python
name = user_input or "guest"
```

Bad when empty string is valid:

``` python
name = config.get("name") or "default"  # may override intentionally empty
```

Use explicit `None` checks when needed.

## 9.3 Guard clauses for expensive operations

``` python
if not authorized:
    return HTTPException(...)
# expensive work after
```

Short-circuit + early return reduces indentation and prevents
unnecessary work.

## 9.4 Boolean-returning APIs

When writing backend libraries: - return `bool` for predicate
functions - avoid returning truthy objects unless you intentionally want
`and/or` chaining semantics

## 9.5 Using `any()` / `all()` correctly

-   `any(iterable)` stops at first truthy element
-   `all(iterable)` stops at first falsy element

They are short-circuiting at the iterator level, which is a real
performance feature for streams/generators.

------------------------------------------------------------------------

# 10) Summary

-   `bool` is a dedicated type but subclasses `int`
-   `True` and `False` are singleton objects in CPython
-   Truthiness uses `__bool__` then `__len__`, else defaults truthy
-   `and/or` return operands and short-circuit via bytecode jumps
-   Production pitfalls: `None` vs `False`, numpy ambiguity, misuse of
    `or` defaults
-   Performance: the cost is usually dispatch/calls, not "branch
    prediction"
