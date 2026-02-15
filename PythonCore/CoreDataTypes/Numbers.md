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

Defined in:
```bash
Include/longobject.h
Objects/longobject.c
```

Internal Structure
```c
typedef struct {
    PyObject_VAR_HEAD
    digit ob_digit[1];
} PyLongObject;
```

Where:
- PyObject_VAR_HEAD → reference count + type pointer + size
- ob_size → number of digits (signed)
- ob_digit[] → array of digits
