# PythonCore → Control Flow → `match / case` (Python 3.10+)

# 0) Quick Orientation: What `match` is

`match` is **structural pattern matching**, not "switch-case".

It matches: - **shape** (sequence, mapping, class attributes) -
**constants** - **types** - and supports **binding** (capturing parts of
a structure)

It is designed to replace: - deep `if/elif` ladders - ad-hoc tuple
unpacking + type checks - brittle nested `get()` chains - some
dispatch-table cases where structure matters

But it is **not**: - regex - generalized unification engine - a drop-in
replacement for dictionary dispatch - a "faster switch" in all cases

------------------------------------------------------------------------

# 1) Language Semantics (PEP 634 Core Rules)

## 1.1 One subject, multiple cases

``` python
match subject:
    case pattern1:
        ...
    case pattern2 if guard:
        ...
    case _:
        ...
```

Execution model:

1)  Evaluate `subject` **once**
2)  Try cases **top-to-bottom**
3)  For each case:
    -   attempt pattern match
    -   if pattern matches and guard (if any) is truthy → execute block,
        stop
4)  If no case matches → do nothing (or fall into `case _` if present)

## 1.2 Patterns can bind variables

``` python
match obj:
    case ("ok", value):
        ...
```

If matches: - `value` becomes a new binding in that case block.

Important: - Bindings occur **only on a successful match** - Failed
cases do not leak partial bindings

## 1.3 Guards (`if` after pattern)

``` python
case ("ok", value) if value > 0:
    ...
```

Guard evaluation happens **after** pattern match succeeds (bindings
exist for the guard).

If guard fails: - That case is considered non-matching - Matching
continues to next case

Guard can have side effects --- be careful (see pitfalls).

------------------------------------------------------------------------

# 2) Pattern Taxonomy (What CPython supports)

## 2.1 Literal / Constant patterns

``` python
case 0:
case "GET":
case None:
case True:
```

### "Constant" is subtle

Names are NOT automatically constants.

-   `case SOME_NAME:` is a **capture pattern** unless qualified (see
    below)
-   To match constants, use:
    -   literals (`"x"`, `123`)
    -   `None/True/False`
    -   qualified names (`Enum.VALUE`, `module.CONST`)

## 2.2 Capture patterns (variable binding)

``` python
case x:
    ...
```

This matches **anything** and binds it to `x`.

This is why `case SOME_NAME:` is dangerous: it captures, it doesn't
compare.

**Rule:** a bare name in a pattern is capture, not constant.

## 2.3 Wildcard pattern `_`

``` python
case _:
    ...
```

Matches anything, binds nothing. This is your "default" case.

## 2.4 OR patterns

``` python
case 400 | 401 | 403:
    ...
```

Matches if any alternative matches.

**Constraint:** all alternatives must bind the same set of names.

Bad:

``` python
case ("ok", x) | ("err", y):  # invalid: different bindings
```

## 2.5 Sequence patterns (tuple/list-like)

``` python
case [a, b]:
case (a, b, c):
case [head, *tail]:
```

Matches **sequences** (not including `str`, `bytes`, `bytearray` by
default).

Key semantics: - Uses the sequence protocol - May perform length
checks - `*rest` captures remaining elements into a list

## 2.6 Mapping patterns (dict-like)

``` python
case {"type": "user", "id": user_id}:
    ...
```

Matches mappings with required keys: - Extra keys are allowed by
default - Values can be patterns themselves - Key lookup uses mapping
protocol (`__getitem__` semantics) - Missing keys → fail

## 2.7 Class patterns (attribute-based)

``` python
case Point(x, y):
    ...
```

This is structural matching via: - `__match_args__` tuple (positional
extraction) - attribute access (keyword extraction)

Important: class patterns **do not call `__iter__`**; they use
attributes.

## 2.8 Type patterns (via class patterns)

``` python
case int(x):
    ...
```

This is NOT "cast". It's a class pattern: - requires
`isinstance(subject, int)` - then binds positional/keyword parts
depending on `__match_args__` (for builtins usually no positional
extraction unless defined)

Often used as:

``` python
case int() as n:
    ...
```

or:

``` python
case int(n):
    ...  # works mainly for custom classes with match args
```

## 2.9 AS patterns (`as` binding)

``` python
case {"id": user_id} as payload:
    ...
```

-   `user_id` captures part
-   `payload` captures the whole subject

## 2.10 Parenthesized / nested patterns

Patterns are compositional:

``` python
case ("ok", {"id": uid, "roles": [*roles]}):
    ...
```

------------------------------------------------------------------------

# 3) Compilation & Runtime (CPython Mechanics)

## 3.1 Subject evaluated once

Unlike repeated `if` checks, `match` evaluates the subject expression
exactly once.

This is a correctness and performance property.

Bad `if` equivalent:

``` python
if expr()["k"] == 1:
    ...
elif expr()["k"] == 2:
    ...
```

`expr()` runs multiple times. In match, subject is computed once and
reused.

## 3.2 How CPython executes a match (high-level)

CPython compiles each case to a series of tests + jumps:

-   Load subject
-   Attempt pattern-specific operations:
    -   type checks (`isinstance`-like)
    -   length checks for sequences
    -   key lookups for mappings
    -   attribute extraction for class patterns
-   If a sub-check fails → jump to next case
-   If all sub-checks pass:
    -   store captured variables
    -   evaluate guard if present
    -   execute body
    -   jump to end of match statement

This looks like a structured cascade of conditional jumps.

## 3.3 Backtracking and partial bindings

CPython ensures bindings only "commit" when a case fully matches.

Implementation strategy conceptually: - use temporary stack / locals for
extracted values - only assign to final locals after case success - on
failure: discard extracted temps and continue

This prevents "half-bound" variables from leaking.

## 3.4 Specialized opcodes (3.10+)

CPython introduced match-related opcodes and support machinery to avoid
building Python-level helper structures for common cases.

You'll see opcodes in `dis` such as: - `MATCH_MAPPING` -
`MATCH_SEQUENCE` - `MATCH_KEYS` - `COPY` - plus conditional jumps that
orchestrate case failure and continuation

(Exact opcode names may vary slightly across 3.10--3.12, but the concept
remains.)

Engineering takeaway: - `match` is not implemented as a library; it's
baked into compiler + VM.

------------------------------------------------------------------------

# 4) Guards: Semantics + Pitfalls

## 4.1 Guard evaluation order

Pattern match happens first, then guard runs using bound variables.

``` python
case {"age": age} if age >= 18:
    ...
```

If `age` doesn't exist, pattern fails and guard never runs.

## 4.2 Guards can hide expensive computations

Bad:

``` python
case data if expensive_check(data):
    ...
```

This is effectively "match anything then guard". You lost structure
matching benefits.

Prefer: - structural patterns first to narrow candidates - guards only
for refinements

## 4.3 Guard side effects

Guards run only when pattern matches, and may run multiple times (for
multiple cases).

Avoid guards with side effects (DB writes, network calls).

------------------------------------------------------------------------

# 5) Critical Pitfalls (Backend Bug Factory)

## 5.1 Name capture vs constant match (MOST COMMON BUG)

``` python
STATUS_OK = 200

match code:
    case STATUS_OK:     # WRONG: this captures into STATUS_OK
        ...
```

This will match anything and bind `STATUS_OK` to `code`, overwriting
your constant name in that scope.

Correct:

``` python
match code:
    case 200:
        ...
```

Or qualify:

``` python
import http
match code:
    case http.HTTPStatus.OK:
        ...
```

Or use Enum:

``` python
case HTTPStatus.OK:
```

(Enums are qualified names, safe.)

**Rule to remember:** bare name = capture.

## 5.2 Irrefutable patterns must be last (or compiler rejects)

A capture pattern like `case x:` matches anything and makes later cases
unreachable.

Similarly `case _:` matches anything.

Place them last.

## 5.3 Sequence matching excludes strings/bytes

Strings are sequences, but match treats them specially (they do not
match sequence patterns by default).

So:

``` python
match "ab":
    case [a, b]:  # won't match like a list
        ...
```

This is intentional to avoid surprising behavior.

## 5.4 Mapping patterns: extra keys allowed

``` python
case {"type": "user"}:
```

Matches `{"type":"user", "extra": 123}` too.

If you need exact keys, you must enforce it: - add guard checking
length - or explicitly match expected keys and reject extras

## 5.5 Class patterns can trigger expensive attribute access

Class patterns do attribute extraction. If properties are computed (or
raise), matching can be expensive or unstable.

Prefer matching on plain dataclasses / simple structs for performance.

## 5.6 OR patterns require consistent captures

``` python
case ("ok", x) | ("ok", y):   # invalid
```

Captures must align.

Use `as` or restructure patterns.

------------------------------------------------------------------------

# 6) Performance Considerations

## 6.1 When match helps

`match` can be clearer and sometimes faster when: - you are matching
nested structures (dicts/tuples) with multiple layers - you need to bind
multiple extracted fields - you want "single evaluation of subject" -
you can avoid repeated attribute lookups / key lookups

## 6.2 When match is not ideal

If you're matching a single scalar against many constants: - a
dictionary dispatch table can be faster and more maintainable - `match`
is still fine, but not automatically superior

Example dispatch:

``` python
handlers = {"GET": get, "POST": post}
handlers.get(method, default)()
```

## 6.3 Cost model (practical)

Pattern matching costs include: - type checks - length checks - key
lookups - attribute loads - building `*rest` lists (allocation) - guard
evaluation

So performance depends on: - complexity of patterns - frequency of
matches - distribution of cases (early match saves work)

As with `if/elif`, order matters for expected frequency.

------------------------------------------------------------------------

# 7) Production Engineering Patterns

## 7.1 HTTP routing / method dispatch

``` python
match method:
    case "GET":
        return handle_get()
    case "POST":
        return handle_post()
    case _:
        raise MethodNotAllowed
```

Use dispatch tables if handlers are numerous or dynamic.

## 7.2 Parsing tagged unions (classic match use-case)

When you receive JSON with a `"type"` field:

``` python
match payload:
    case {"type": "user", "id": int(uid)}:
        return User(uid)
    case {"type": "order", "id": int(oid), "amount": amt} if amt > 0:
        return Order(oid, amt)
    case _:
        raise ValueError("Invalid payload")
```

This reads like a declarative schema.

## 7.3 Result types (`("ok", value)` / `("err", reason)`)

``` python
match result:
    case ("ok", value):
        ...
    case ("err", reason):
        ...
```

This replaces brittle tuple unpacking + if checks.

## 7.4 Dataclasses + class patterns

With dataclasses:

``` python
from dataclasses import dataclass

@dataclass
class Point:
    x: int
    y: int

match obj:
    case Point(x=0, y=y):
        ...
```

Be mindful: - attribute access cost - property side effects (avoid)

## 7.5 Enforcing exact dict shape

``` python
match payload:
    case {"type": "user", "id": uid} as p if len(p) == 2:
        ...
```

This rejects extra keys explicitly.

------------------------------------------------------------------------

# 8) Interview-Level Traps

1)  Bare names in patterns **capture**; they are not constants
2)  `case _` and `case x` are irrefutable → must be last
3)  `match` subject evaluated exactly once
4)  `case ... if guard` runs guard after successful match
5)  Mapping patterns allow extra keys
6)  OR patterns must capture the same names
7)  Strings don't behave like list sequence patterns in matching

------------------------------------------------------------------------

# 9) Mental Model Cheat Sheet

-   Think: "try pattern, jump on failure"
-   `elif` == nested `if` inside `else`
-   `match` == compiled cascade of tests + binding + optional guard
-   capture names are **bindings**, not comparisons
-   design your patterns to keep extraction cheap and deterministic
