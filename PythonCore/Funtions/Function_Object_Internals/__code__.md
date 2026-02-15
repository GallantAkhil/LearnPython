# PythonCore → Functions (Advanced) → Function Object Internals: `__code__`

# 1) Function Object vs Code Object

A Python function is a runtime object holding: - a reference to a `code`
object (`__code__`) - globals (`__globals__`) - defaults
(`__defaults__`, `__kwdefaults__`) - closure (`__closure__`) - metadata
(`__name__`, `__qualname__`, `__annotations__`, etc.)

The `code` object is the compiled representation: - bytecode -
constants - variable names - argument counts - flags

Multiple functions can share the same code object in some patterns, but
typically each `def` produces one code object.

------------------------------------------------------------------------

# 2) Key `__code__` Fields (What Matters)

Important fields (names stable across versions, but details evolve):

-   `co_name` / `co_qualname`: naming
-   `co_filename`: origin file
-   `co_firstlineno`: starting line
-   `co_consts`: constants used
-   `co_names`: global names referenced
-   `co_varnames`: local variables (including arguments)
-   `co_freevars`: free variables captured from outer scopes
-   `co_cellvars`: variables that become cells for inner closures

Argument interface: - `co_argcount`: positional-or-keyword count (not
including pos-only) - `co_posonlyargcount`: positional-only count -
`co_kwonlyargcount`: keyword-only count

Execution flags: - `co_flags`: generator/coroutine/varargs/varkwargs
etc.

------------------------------------------------------------------------

# 3) Bytecode Relationship

The code object contains bytecode executed by CPython's VM.

You can inspect via `dis`:

``` python
import dis
dis.dis(fn)
```

Engineering use: - diagnose performance issues (global lookups,
attribute loads) - verify short-circuit structure - reason about
comprehension/lambda expansion - confirm generator/coroutine behavior

------------------------------------------------------------------------

# 4) CPython Runtime Use

When a Python function is called: - CPython creates a frame with a
pointer to `__code__` - uses counts in `__code__` for argument binding -
executes bytecode sequentially with jump opcodes, stack operations, and
calls

Thus `__code__` is the blueprint for execution.

------------------------------------------------------------------------

# 5) Practical Engineering Patterns

## 5.1 Signature tools and frameworks

Frameworks often rely on: - `inspect.signature` - annotations - and the
underlying code arg counts

If decorators break these, dependency injection and validation can fail.

## 5.2 Debug tooling

You can trace unexpected behavior by inspecting: - `co_consts` for
embedded constants - `co_names` for referenced globals - `co_freevars`
for closure dependencies

## 5.3 Performance audits

Disassembly reveals: - repeated global loads - repeated attribute
lookups - opportunities to hoist locals or use builtins

------------------------------------------------------------------------

# 6) Pitfalls

-   Code objects are not a stable ABI across Python versions for
    low-level manipulation
-   Don't build production logic that depends on exact opcode sequences
-   Treat introspection output as diagnostic, not contractual

Security note: - dynamic code execution (`eval/exec`) creates code
objects; be extremely careful with untrusted input.

------------------------------------------------------------------------

# 7) Interview Traps

1)  `__code__` is compiled bytecode + metadata, separate from function
    object
2)  Argument counts live on `__code__` (`co_posonlyargcount`,
    `co_kwonlyargcount`)
3)  Closures relate to `co_freevars` and `co_cellvars`
4)  Bytecode inspection with `dis` is often used for debugging

------------------------------------------------------------------------

# 8) Summary

`__code__` is the compiled execution blueprint for a function: argument
layout, locals, constants, free variables, flags, and bytecode. Knowing
it gives you CPython-level insight into performance, correctness, and
framework behavior.

------------------------------------------------------------------------
