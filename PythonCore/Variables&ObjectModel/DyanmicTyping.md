# PythonCore → Variables & Object Model → Dynamic Typing

**Systems-Level Deep Dive (object model + name binding + runtime type
semantics + performance)**

------------------------------------------------------------------------

## 1) Conceptual Model

Python is dynamically typed, meaning:

-   Variables are **names**
-   Names bind to **objects**
-   Objects carry type information
-   Types are checked at runtime

There is no "typed variable", only "typed object".

``` python
x = 10
x = "hello"   # name rebound to new object
```

------------------------------------------------------------------------

## 2) CPython Object Model

Every object begins with:

``` c
typedef struct {
    Py_ssize_t ob_refcnt;
    PyTypeObject *ob_type;
} PyObject;
```

So every object has: - reference count - pointer to its type object

Type lives with the object, not the name.

------------------------------------------------------------------------

## 3) Name Binding Mechanics

Assignment does NOT copy objects:

``` python
a = [1,2]
b = a
```

Both names reference the same object (pointer copy).

Rebinding:

``` python
a = [3,4]   # b unaffected
```

------------------------------------------------------------------------

## 4) Runtime Type Resolution

Operations dispatch via type slots:

``` python
a + b
```

Is resolved as: - lookup of `nb_add` slot in type object - dynamic
dispatch at runtime

No compile-time type specialization (except recent adaptive interpreter
optimizations).

------------------------------------------------------------------------

## 5) Performance Implications

Dynamic typing costs: - runtime dispatch - boxing of primitives -
reference counting

Hot loops benefit from: - local variable binding - using builtins
(C-level loops) - avoiding repeated attribute lookups

------------------------------------------------------------------------

## 6) Engineering Patterns

-   Use duck typing for flexibility
-   Validate at boundaries (Pydantic, type hints + runtime checks)
-   Avoid unnecessary isinstance checks internally
-   Prefer protocols over concrete types

------------------------------------------------------------------------

## 7) Interview Traps

1)  Variables are references, not containers
2)  Type lives on object
3)  Assignment never copies (unless explicit copy)
4)  Dynamic dispatch happens at runtime

------------------------------------------------------------------------
