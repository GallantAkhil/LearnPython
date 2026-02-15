# PythonCore → Functions (Advanced) → Keyword-only Parameters (`*`)

# 1) Conceptual Model

A **keyword-only** parameter must be provided by keyword, never by
position.

Introduced in Python 3 via PEP 3102.

Syntax:

``` python
def f(a, b, *, c, d=10):
    ...
```

Rules: - `a`, `b` are positional-or-keyword - `c`, `d` are
keyword-only - `*` acts as a "positional argument stopper"

Examples:

``` python
f(1, 2, c=3)       # OK
f(1, 2, 3)         # TypeError: c passed positionally
```

If you already have `*args`, everything after is keyword-only:

``` python
def g(*args, c, d=10):
    ...
```

------------------------------------------------------------------------

# 2) Why Keyword-only Exists (API Design)

## 2.1 Clarity and self-documentation

Keyword-only makes call sites explicit:

``` python
request(url, timeout=5, retries=3)
```

Better than hidden positional meaning like `request(url, 5, 3)`.

## 2.2 Prevents accidental positional misuse

You can add keyword-only parameters later without breaking existing
positional call sites.

## 2.3 Safer defaults and tri-state behavior

Keyword-only parameters often pair with sentinels and `None` to
distinguish "unset" vs "explicit None".

------------------------------------------------------------------------

# 3) Binding Semantics (Exact Rules)

Given:

``` python
def f(a, b=2, *, c, d=4):
    ...
```

-   `a` required positional-or-keyword
-   `b` optional positional-or-keyword
-   `c` required keyword-only
-   `d` optional keyword-only

Valid calls:

``` python
f(1, c=3)
f(a=1, c=3, d=10)
```

Invalid:

``` python
f(1, 2, 3)       # c passed positionally
f(1)             # missing c
```

------------------------------------------------------------------------

# 4) CPython Internals: Where Keyword-only Lives

The code object stores counts: - `co_kwonlyargcount` for keyword-only
args - mapping occurs during call setup into fast locals

Keyword-only args are allocated slots in the local variable array just
like others, but: - they are not filled by positional inputs - they are
filled only via keyword mapping or defaults

Error modes include: - missing required keyword-only argument -
unexpected keyword argument - duplicate values for argument
(positional + keyword)

------------------------------------------------------------------------

# 5) Interactions with `*args` / `**kwargs`

## 5.1 `*args` makes following parameters keyword-only

``` python
def f(*args, sep=","):
    ...
```

Callers must do:

``` python
f(1, 2, 3, sep="|")
```

## 5.2 `**kwargs` collects extra keywords, not keyword-only

Keyword-only parameters are still explicit; `**kwargs` collects only
remaining unmatched keywords.

``` python
def f(*, a, **kwargs):
    ...
```

-   `a` must be present
-   extras go into `kwargs`

------------------------------------------------------------------------

# 6) Production Patterns

## 6.1 Make "configuration-like" parameters keyword-only

For backend APIs, configs should be keyword-only to prevent ordering
bugs:

``` python
def connect(host, port, *, timeout=5, ssl=True):
    ...
```

## 6.2 Add new features safely

Adding `*, feature_flag=False` won't break older callers who passed only
positional args.

## 6.3 Enforce semantic correctness

Use keyword-only for parameters that are easy to confuse: -
`start/end` - `min/max` - `timeout/retries` - `limit/offset`

------------------------------------------------------------------------

# 7) Pitfalls

## 7.1 Overusing keyword-only can make call sites verbose

Use it where it improves clarity and stability.

## 7.2 Mixing `None` as both "unset" and "meaningful"

If `None` is valid input, use a sentinel for "not provided".

## 7.3 Wrappers that lose signature

Decorators with `*args, **kwargs` can accidentally accept positional
passing that original would reject. Preserve signature when correctness
matters.

------------------------------------------------------------------------

# 8) Interview Traps

1)  `*` is the keyword-only barrier
2)  Params after `*args` are keyword-only
3)  Keyword-only allows adding params without breaking positional
    callers
4)  Keyword-only is about API design, not just syntax

------------------------------------------------------------------------

# 9) Summary

-   Keyword-only parameters enforce explicit naming at call sites
-   CPython maps them via keyword dictionary → fast locals slots
-   Great for config parameters, forward-compatible APIs, and reducing
    misuse

------------------------------------------------------------------------

