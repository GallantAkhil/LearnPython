# Numbers

Python’s numeric system is not just a few types. It is a carefully engineered hierarchy with different runtime representations, memory layouts, and arithmetic algorithms.

--

## Conceptual Model

### Python numeric tower

int        → arbitrary precision integer
float      → IEEE-754 double precision
decimal    → base-10 arbitrary precision
fractions  → rational numbers
complex    → pair of floats

Hierarchy (from numbers module ABCs):

Number
 ├── Complex
 │    ├── Real
 │    │    ├── Rational
 │    │    │    └── Integral

