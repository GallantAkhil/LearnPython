# PythonCore → Functions (Advanced) → First-class Functions

# 1) Conceptual Model

"First-class" means functions can be:

-   assigned to variables
-   passed as arguments
-   returned from other functions
-   stored in data structures

``` python
def add(x, y): return x + y
op = add
op(1, 2)  # 3
```

------------------------------------------------------------------------

# 2) Callables in Python

A callable is any object that can be invoked with `()`.

Major categories:

-   function objects (`def`, `lambda`)
-   bound/unbound methods
-   classes (calling constructs instances via `__new__`/`__init__`)
-   instances implementing `__call__`
-   builtins (C functions)

Check:

``` python
callable(obj)
```

------------------------------------------------------------------------

# 3) CPython Invocation (High-level)

At runtime, calling does:

1)  Evaluate callable object
2)  Evaluate arguments
3)  Build / pass arguments via internal calling convention (often
    vectorcall)
4)  Invoke the callable's call slot (C-level function pointer)

Different callable types have different internal call slots: - Python
function: creates frame and runs bytecode - Builtin function: calls C
implementation directly - Bound method: injects `self` then calls
function - Callable instance: resolves `__call__` then calls it

Engineering implication: \> Calling pure Python functions is much more
expensive than calling a C builtin, because of frame + bytecode
execution overhead.

------------------------------------------------------------------------

# 4) Practical Engineering Patterns

## 4.1 Dispatch tables (replace if/elif ladders)

``` python
handlers = {
    "create": create_user,
    "delete": delete_user,
}
handlers[action](payload)
```

Benefits: - O(1) lookup - easier extensibility - cleaner code review

## 4.2 Dependency injection

Pass behavior explicitly:

``` python
def service(repo_get):
    return repo_get("id")
```

This improves testability and decoupling.

## 4.3 Callbacks and hooks

Framework design often uses callbacks (middleware, lifecycle hooks). Use
typed protocols for clarity.

------------------------------------------------------------------------

# 5) Pitfalls

## 5.1 Late binding in closures (common)

Functions capture variables by reference, not value. See closures deep
dive.

## 5.2 Introspection loss

Wrappers hide original signatures unless you use `functools.wraps` and
possibly signature preservation tooling.

## 5.3 Overusing functional style can hurt readability

Use higher-order functions where they clarify; avoid "clever" one-liners
in production.

------------------------------------------------------------------------

# 6) Performance Notes

-   Function calls are expensive; minimize calls in hot loops
-   Prefer builtins and vectorized operations (C-level loops) when
    possible
-   Local variable binding reduces global lookup overhead inside tight
    loops

------------------------------------------------------------------------

# 7) Interview Traps

1)  Classes are callable
2)  Instances with `__call__` are callable
3)  Bound methods capture `self` at access time
4)  Python function calls are costlier than builtins

------------------------------------------------------------------------

# 8) Summary

Functions are first-class objects enabling powerful composition and
clean architecture, but you must manage: - call overhead - closure
binding behavior - introspection and signatures - API clarity and
maintainability

------------------------------------------------------------------------
