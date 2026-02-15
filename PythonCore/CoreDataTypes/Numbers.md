# Numbers

Python’s numeric system is not just a few types. It is a carefully engineered hierarchy with different runtime representations, memory layouts, and arithmetic algorithms.

--

# 1️⃣ Conceptual Model

### Python numeric tower

Number
 ├── Complex
 │    ├── Real
 │    │    ├── Rational
 │    │    │    └── Integral

### Concrete types

| Type      | Category   | Precision Model |
|------------|------------|----------------|
| `int`      | Integral   | Arbitrary precision |
| `float`    | Real       | IEEE-754 double |
| `Decimal`  | Real       | Base-10 arbitrary precision |
| `Fraction` | Rational   | Exact rational |
| `complex`  | Complex    | Pair of floats |

Design philosophy:
- `int` prioritizes correctness (no overflow)
- `float` prioritizes hardware speed
- `Decimal` prioritizes financial correctness
- `Fraction` prioritizes exactness

# 2️⃣ `int` — Arbitrary Precision Integer

## Conceptual Model

Python integers:
- Unbounded
- No overflow
- Limited only by available memory

```python
2 ** 10_000_000  # valid if RAM allows
```
Unlike C:
- No 32-bit or 64-bit overflow
- No wraparound

## CPython Internal Implementation

### Defined in:
```bash
Include/longobject.h
Objects/longobject.c
```

### Internal Structure
```c
typedef struct {
    PyObject_VAR_HEAD
    digit ob_digit[1];
} PyLongObject;
```

### Where:
- PyObject_VAR_HEAD → reference count + type pointer + size
- ob_size → number of digits (signed)
- ob_digit[] → array of digits

## Internal Representation

### On 64-bit CPython:

```python
Base = 2^30
```
Each digit = 30 bits (not 32).

Why 30 bits?
- Prevent overflow during internal multiplication
- Leave headroom for carry operations

### Number stored as:
```ini
value = Σ (ob_digit[i] * 2^(30*i))
```
Sign is stored separately in ob_size.

## Algorithm Switching

CPython dynamically switches multiplication algorithms:

Size Range |	Algorithm |
|------------|------------|
Small	| Schoolbook O(n²) |
Medium	| Karatsuba O(n^1.585) |
Larger	| Toom-Cook |
Very Large	| FFT-based |

This makes Python surprisingly efficient for very large integers.

## Small Integer Caching

CPython preallocates integers:
```python
[-5, 256]
```
Example:
```python
a = 100
b = 100
a is b  # True
```
But:
```python
a = 1000
b = 1000
a is b  # Usually False
```
Why?
- Small integers are interned at startup.
- Others are heap-allocated.

Runtime Behavior
- Every arithmetic operation creates a new PyLongObject.
- Integers are immutable.
- Large integers allocate memory dynamically.

## Edge Cases
### Bitwise on Negative Integers

Python simulates infinite two’s complement.
```python
~(-1)  # 0
```
Internally:
- Stored as sign + magnitude
- But operations mimic infinite sign extension

### Floor Division Behavior
```python
5 // -2  # -3
```
Python uses mathematical floor, not truncation.
C would return -2.

## Performance Considerations
- Big integers scale with digit count
- Heavy integer loops are slow in pure Python
- Each operation:
 - Allocates memory
 - Performs dynamic dispatch
 - Updates refcount
Engineering implication:
- For heavy numeric workloads → use NumPy or C extensions.

# 3️⃣ float — IEEE-754 Double Precision
## Conceptual Model
Python float = C double (64-bit IEEE 754).

Structure:
| Field |	Bits |
|-------|------|
| Sign	| 1 |
| Exponent	| 11 |
| Mantissa	| 52 |

## CPython Structure

Defined in:
```bash
Include/floatobject.h
Objects/floatobject.c
```
```c
typedef struct {
    PyObject_HEAD
    double ob_fval;
} PyFloatObject;
```

Memory footprint (64-bit):
- ~24 bytes per float
Massive overhead compared to C.

## Precision Reality
```python
0.1 + 0.2 != 0.3
```

Because:
- 0.1 is not exactly representable in binary
- Mantissa is 52 bits (~15 decimal digits precision)

## Edge Cases
###NaN
```python
float('nan') == float('nan')  # False
```

IEEE rule:
- NaN != NaN
Danger in:
- Sets
- Dict keys
- Equality comparisons

### Signed Zero
```python
0.0 == -0.0  # True
```
But:
```python
import math
math.copysign(1, -0.0)  # -1.0
```
Sign bit preserved internally.

Performance
- Fast (delegated to CPU FPU)
- No arbitrary precision
- Ideal for ML / scientific computing

# 4️⃣ decimal.Decimal
## Conceptual Model
Base-10 floating point with arbitrary precision.
Representation:
```ini
value = sign × coefficient × 10^exponent
```
Backed by:
- libmpdec (C library)

## Internal Behavior
Decimal arithmetic is governed by a context:
```python
from decimal import getcontext
getcontext().prec = 50
```

Context controls:
- Precision
- Rounding
- Traps
- Overflow rules
Context is thread-local.

## Edge Cases
- Mixing with float → TypeError
- Decimal(0.1) imports float error
- Always use:
```python
Decimal("0.1")
```

## Performance
- Much slower than float
- Large memory footprint (~100+ bytes per instance)

Use only for:
- Finance
- Monetary calculations


# 5️⃣ fractions.Fraction
## Conceptual Model
Exact rational number:
```python
Fraction(1, 3)
```

Stored as:
```c
numerator (int)
denominator (int)
```
Always reduced via GCD.

## Internal Mechanics

On creation:
```c
g = gcd(numerator, denominator)
numerator //= g
denominator //= g
```
GCD computation is expensive.

## Performance
- Very slow
- Memory heavy
- Not suitable for high-performance systems

Use only for:
- Symbolic math
- Exact ratio modeling
