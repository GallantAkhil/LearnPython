# PythonCore → Datatypes → `None` (`NoneType`)

# 1) Conceptual Model

### 1.1 `None` represents "no value / not provided / missing / unknown"

Python uses `None` as the universal placeholder for "absence of a
meaningful value".\
It is intentionally distinct from:

-   `0` (a numeric value)
-   `""` (an empty string)
-   `False` (boolean negation)
-   `[]` / `{}` (empty containers)

### 1.2 `None` is a singleton object of type `NoneType`

There is exactly one `None` in a Python process.

``` python
x = None
y = None
x is y  # True (always)
```

### 1.3 Why singleton matters

Singleton semantics allow:

-   Identity checks (`is`) to be correct and fast
-   Stable hashing & dictionary usage
-   A reliable sentinel for "missing" that cannot be constructed
    accidentally

------------------------------------------------------------------------

# 2) CPython Internal Implementation

CPython represents `None` as a static singleton:

-   `Py_None` points to a global `_Py_NoneStruct`
-   That object's type is `NoneType`

This mirrors `True/False` behavior (also singletons), but `None` is even
more fundamental.

### 2.1 Memory behavior

-   `None` is allocated once and lives for the lifetime of the
    interpreter
-   No per-call allocations
-   Returning `None` from a function returns the same singleton

### 2.2 Refcount behavior

`None` is referenced everywhere. CPython ensures:

-   It is never freed
-   It is safe to INCREF/DECREF like any other object

Practically: - `None` behaves like a normal PyObject at the refcount API
level - But it is immortal for program lifetime

------------------------------------------------------------------------

# 3) Runtime Semantics of `None`

## 3.1 Truthiness

`None` is falsy.

``` python
bool(None)  # False
```

This matters in control flow:

``` python
if value:
    ...
```

Pitfall: This treats `None` the same as `0`, `""`, `[]`, `{}`, `False`.

If those distinctions matter, be explicit:

``` python
if value is None:
    ...
```

------------------------------------------------------------------------

## 3.2 Comparisons (`None` with others)

In Python 3: - Ordering comparisons between unrelated types raise
`TypeError`

``` python
None < 0   # TypeError
None > ""  # TypeError
```

Equality:

``` python
None == None  # True
None == 0     # False
None == ""    # False
```

Identity is preferred:

``` python
x is None
x is not None
```

Why? - `==` can be overloaded by user types (rare for None, but possible
patterns exist) - `is` is constant-time identity check and semantically
correct for singletons

------------------------------------------------------------------------

## 3.3 Hashing

`None` is hashable.

``` python
hash(None)
```

It can be used as a dict key or set element.

Implications: - Many caches store `None` meaningfully - This can create
bugs when `None` means both "cached negative result" and "not in cache"

This leads to sentinel patterns (covered later).

------------------------------------------------------------------------

## 3.4 Attribute access & method calls

`None` has essentially no behavior except identity, repr, and
truthiness.

Common runtime failure mode:

``` python
None.some_attr  # AttributeError
```

Backend insight: - Most "NoneType has no attribute" errors are upstream
contract bugs. - Fix is not to sprinkle `if x is None` everywhere; fix
invariants and typing.

------------------------------------------------------------------------

# 4) `None` as Default Return Value

In Python, a function with no explicit return returns `None`.

``` python
def f():
    pass

f() is None  # True
```

Production implication: - Many bugs come from missing `return`
statements. - Especially in FastAPI dependencies and validators.

Example:

``` python
def authorize(user):
    if user.is_admin:
        return True
    # forgot return False -> returns None
```

Then:

``` python
if authorize(user):
    ...
```

`None` behaves as falsy, masking the bug until later.

Better: - Always return explicit booleans for predicates. - Use type
checkers (mypy/pyright) to catch missing returns.

------------------------------------------------------------------------

# 5) `None` in Parameter Defaults (Huge Pitfall Category)

## 5.1 Safe default pattern

Python default arguments are evaluated **once** at function definition
time.

So, mutable defaults are dangerous:

``` python
def f(x, cache={}):  # BAD
    ...
```

Correct pattern:

``` python
def f(x, cache=None):
    if cache is None:
        cache = {}
    ...
```

`None` is used here as a sentinel for "not provided".

------------------------------------------------------------------------

## 5.2 But `None` is sometimes an allowed value

If `None` is a *valid* user input distinct from "missing", then using it
as sentinel breaks.

Example: - `timeout=None` meaning "no timeout" is valid - `timeout`
omitted meaning "use default timeout" is different

In this case, **use a unique sentinel object**:

``` python
_MISSING = object()

def request(timeout=_MISSING):
    if timeout is _MISSING:
        timeout = DEFAULT_TIMEOUT
    ...
```

This is the **real sentinel pattern**.

------------------------------------------------------------------------

# 6) `None` at System Boundaries (JSON, DB, APIs)

## 6.1 JSON mapping

-   Python `None` ↔ JSON `null`

But beware: - Missing key ≠ key present with null

Example payloads:

``` json
{}
{"name": null}
```

These are different semantics in many APIs.

Backend pattern: - Treat "missing" and "null" as separate states when
patching resources.

## 6.2 Database NULL mapping

-   SQL NULL semantics are 3-valued logic
-   `NULL = NULL` is not true; it is UNKNOWN

ORMs map NULL to Python `None`, but SQL comparisons require `IS NULL`.

Engineering implication: - Always use ORM-safe operators (`is_(None)` in
SQLAlchemy, etc.)

## 6.3 Cache semantics

`None` can mean: - cached negative result (e.g., "user not found") -
absence of cache entry

If you use `dict.get(k)` and store `None`, you can't tell those apart.

Correct pattern:

``` python
_MISSING = object()

val = cache.get(key, _MISSING)
if val is _MISSING:
    val = compute()
    cache[key] = val
```

------------------------------------------------------------------------

# 7) Tri-State Logic (True/False/None)

Many backend domains have tri-state fields:

-   feature flag: enabled / disabled / unspecified
-   consent: yes / no / not asked
-   validation: pass / fail / not run

Correct handling:

``` python
if flag is True:
    ...
elif flag is False:
    ...
else:
    ...  # None (unknown)
```

Danger of:

``` python
if flag:
    ...
```

This collapses False and None.

------------------------------------------------------------------------

# 8) Common Pitfalls

## 8.1 `== None` instead of `is None`

``` python
x == None  # discouraged
```

Use:

``` python
x is None
```

Reason: - semantic correctness (singleton) - speed (identity) - avoids
overloaded equality edge cases

## 8.2 `if not x` hides bugs

``` python
if not x:
    ...
```

This treats `None`, `0`, `""`, `[]`, `{}`, `False` the same.

Fix: pick intent: - emptiness check: `if not items:` - missing check:
`if x is None:`

## 8.3 Returning `None` where bool expected

Predicates should return bool, not None:

``` python
def is_valid(x) -> bool:
    ...
```

Static typing helps prevent accidental `None` returns.

## 8.4 Optional chaining instincts (not Python)

Python does not support `?.` like JS.\
So code often becomes messy with None checks.

Engineering response: - enforce invariants early - use guard clauses -
use Pydantic / typing to validate at boundaries

------------------------------------------------------------------------

# 9) Performance Considerations

## 9.1 `is None` is extremely fast

Identity comparison is pointer comparison.

``` python
x is None
```

This is as cheap as it gets in Python-level checks.

## 9.2 The expensive part is not `None`, it's the checks everywhere

Over-checking for None in hot paths: - increases branching and bytecode
count - creates clutter - indicates unclear contracts

Performance pattern: - validate once at boundary - convert to
non-optional internal representation - keep core logic free of repeated
None checks

------------------------------------------------------------------------

# 10) Production Engineering Patterns (Backend Grade)

## 10.1 Use `None` for "missing" only when `None` is not a valid value

-   Great for "optional argument not provided"
-   Great for "no DB value"
-   Great for "not computed yet"

But if `None` is meaningful input, use a sentinel object.

## 10.2 Sentinel object pattern (recommended)

``` python
_MISSING = object()

def f(x=_MISSING):
    if x is _MISSING:
        x = compute_default()
    return x
```

This avoids conflating "missing" with "explicit None".

## 10.3 API Patch semantics (PUT/PATCH)

For PATCH-style updates: - missing field → do not change - present null
(`None`) → set to null

Implementation approach: - distinguish "field absent" from "field
present with None" in validation layer

## 10.4 Cache negative results safely

Use sentinel to distinguish "no key" from "key mapped to None".

## 10.5 Typing: `Optional[T]` should be eliminated internally

Pattern: - At boundary: `Optional[str]` - Inside core: `str` with
invariant "never None" enforced upfront

This reduces downstream complexity.

------------------------------------------------------------------------

# 11) Interview-Level Traps

1)  `None` is a singleton → use `is None`
2)  `None` is falsy but not equal to `False`
3)  `if not x` collapses None/0/""/\[\] etc.
4)  default args are evaluated once → `None` as safe default sentinel
5)  `None` vs missing key in JSON are different semantics
6)  `dict.get()` cannot distinguish missing key from stored None without
    sentinel
7)  Python 3 forbids ordering comparisons with None across unrelated
    types

------------------------------------------------------------------------

# 12) Summary

-   `None` is a **singleton** (`NoneType`) used to represent absence
-   In CPython it is a **static immortal object** (`Py_None`)
-   Truthiness: `None` is falsy, but don't confuse falsy with missing
-   Use `is None` / `is not None`
-   Use **sentinel objects** when None is a valid value
-   Be explicit at API/DB/cache boundaries to preserve semantics


**End of document.**
