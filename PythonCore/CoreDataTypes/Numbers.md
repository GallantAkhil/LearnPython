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
 -- Allocates memory
 -- Performs dynamic dispatch
 -- Updates refcount
Engineering implication:
- For heavy numeric workloads → use NumPy or C extensions.
