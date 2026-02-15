# PythonCore → Control Flow → `if / elif / else`

# 1) Conceptual Model

`if/elif/else` is Python's conditional branching construct.

High-level behavior:

``` python
if condition1:
    block1
elif condition2:
    block2
else:
    block3
```

Execution rules:

1)  Evaluate `condition1`
2)  If truthy → execute `block1`, skip rest
3)  Else evaluate `condition2`
4)  If truthy → execute `block2`, skip rest
5)  Else execute `block3`

Only one branch executes.

------------------------------------------------------------------------

# 2) Compilation Model (AST Level)

Python does not interpret source directly.

Pipeline:

1)  Source → Tokenizer
2)  Tokens → AST
3)  AST → Bytecode
4)  Bytecode → CPython Virtual Machine

`if` becomes an `If` AST node containing:

-   test expression
-   body
-   orelse (which may contain nested If nodes for elif chains)

Important:

`elif` is syntactic sugar.

Internally:

``` python
if A:
    ...
elif B:
    ...
```

is compiled as nested:

``` python
if A:
    ...
else:
    if B:
        ...
```

------------------------------------------------------------------------

# 3) Bytecode-Level Execution

Let's examine conceptual bytecode for:

``` python
if x:
    y = 1
else:
    y = 2
```

High-level disassembly behavior:

1)  LOAD_FAST x
2)  POP_JUMP_IF_FALSE label_else
3)  LOAD_CONST 1
4)  STORE_FAST y
5)  JUMP_FORWARD label_end label_else:
6)  LOAD_CONST 2
7)  STORE_FAST y label_end:

Key points:

-   CPython evaluates expression first
-   Converts to truth value via `PyObject_IsTrue`
-   Uses conditional jump opcode
-   Only executes one branch

------------------------------------------------------------------------

# 4) Truthiness Integration

When evaluating condition:

``` python
if obj:
```

CPython calls:

-   `__bool__()` if defined
-   else `__len__()`
-   else default truthy

Therefore:

Cost of `if` depends on cost of truthiness evaluation.

If `__len__()` is expensive, `if obj:` is expensive.

------------------------------------------------------------------------

# 5) Short-Circuiting in Conditions

Compound conditions:

``` python
if A and B:
```

Bytecode:

1)  Evaluate A
2)  If falsy → jump to end
3)  Else evaluate B

This ensures B is only evaluated when needed.

Same for `or`, reversed logic.

This is critical for guard patterns:

``` python
if user and user.is_admin:
```

If `user` is None, second expression never runs.

------------------------------------------------------------------------

# 6) Performance Considerations

## 6.1 Branching cost in Python

Unlike C/C++:

-   CPU branch prediction is negligible factor
-   Dominant cost = Python bytecode dispatch + object model

So micro-optimizing branch ordering rarely matters unless inside tight
loops.

## 6.2 Condition Re-evaluation Cost

Bad:

``` python
if expensive_call():
    ...
elif expensive_call():
    ...
```

This calls function twice.

Better:

``` python
result = expensive_call()
if result:
    ...
elif other_condition:
    ...
```

Cache expensive results.

## 6.3 Avoid deep nesting

Deep nesting increases:

-   cognitive load
-   bug probability
-   branch misinterpretation

Prefer guard clauses:

``` python
if not condition:
    return error

# happy path continues
```

------------------------------------------------------------------------

# 7) Common Backend Pitfalls

## 7.1 Using `if not x` when meaning `x is None`

``` python
if not timeout:
```

This treats 0, None, False the same.

Use explicit intent:

``` python
if timeout is None:
```

## 7.2 Ordering conditions incorrectly

Example:

``` python
if x > 0:
    ...
elif x > 10:
    ...
```

Second condition unreachable.

Always order from most specific to most general.

## 7.3 Shadowed logic in elif chains

Long `elif` chains often indicate missing abstraction.

Better patterns:

-   Dictionary dispatch
-   Strategy pattern
-   Match-case (Python 3.10+)

------------------------------------------------------------------------

# 8) Structural Patterns

## 8.1 Guard Clause Pattern

Preferred in backend code:

``` python
def process(user):
    if user is None:
        raise ValueError("User required")

    if not user.active:
        return "inactive"

    # core logic here
```

Reduces indentation and improves clarity.

------------------------------------------------------------------------

## 8.2 Dispatch Table Pattern

Instead of:

``` python
if action == "create":
    ...
elif action == "update":
    ...
elif action == "delete":
    ...
```

Use:

``` python
handlers = {
    "create": handle_create,
    "update": handle_update,
    "delete": handle_delete,
}

handler = handlers.get(action)
if handler is None:
    raise ValueError("Unknown action")

handler()
```

Scales better and is more maintainable.

------------------------------------------------------------------------

# 9) Control Flow and Exceptions

Sometimes better to use exceptions than nested conditionals:

Instead of:

``` python
if not valid:
    return error
```

In deep call stacks, use:

``` python
if not valid:
    raise ValidationError
```

Then handle at boundary.

This simplifies core logic.

------------------------------------------------------------------------

# 10) Edge Cases

## 10.1 Assignment inside condition (Walrus operator)

Python 3.8+:

``` python
if (n := len(data)) > 10:
    ...
```

Semantics:

-   Evaluate expression
-   Assign result
-   Compare
-   Branch

Compiled into:

-   LOAD
-   STORE
-   COMPARE
-   JUMP

Use carefully; improves performance when avoiding repeated calls.

------------------------------------------------------------------------

## 10.2 Empty `else` blocks

Avoid:

``` python
if cond:
    ...
else:
    pass
```

Indicates incomplete logic or unnecessary branch.

------------------------------------------------------------------------

## 10.3 Dangling else confusion (not possible in Python)

Due to indentation, Python avoids classic C dangling-else ambiguity.

------------------------------------------------------------------------

# 11) Interview-Level Traps

1)  `elif` is syntactic sugar for nested `if` in `else`
2)  Only one branch executes
3)  Conditions are evaluated top to bottom
4)  Truthiness protocol applies
5)  Short-circuit prevents evaluation of later expressions
6)  Assignment expressions evaluate once
7)  Order of conditions matters for correctness

------------------------------------------------------------------------

# 12) Performance Summary

-   Condition evaluation cost depends on truthiness implementation
-   Python branching overhead is dominated by interpreter dispatch
-   Avoid repeated expensive expressions
-   Prefer guard clauses
-   Avoid deeply nested blocks

------------------------------------------------------------------------

# 13) Production Engineering Checklist

When writing backend conditionals:

-   Be explicit about None vs falsy
-   Cache expensive expressions
-   Prefer guard clauses
-   Replace long elif chains with dispatch tables
-   Keep branch logic flat
-   Validate at boundary, simplify core logic


**End of document.**
