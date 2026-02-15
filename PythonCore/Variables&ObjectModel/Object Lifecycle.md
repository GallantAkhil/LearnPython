# PythonCore → Variables & Object Model → Object Lifecycle

**Systems-Level Deep Dive (`__new__`, `__init__`, reference lifecycle,
destruction)**

------------------------------------------------------------------------

## 1) Creation

Object creation steps:

1)  `__new__` allocates memory
2)  `__init__` initializes state

For immutable types, customization often happens in `__new__`.

------------------------------------------------------------------------

## 2) Lifetime

Object exists as long as references \> 0.

Names are bindings; deleting name decreases refcount.

``` python
del x
```

------------------------------------------------------------------------

## 3) Destruction

When refcount reaches zero: - `__del__` called (if exists) - memory
freed

But not deterministic in cyclic GC scenarios.

------------------------------------------------------------------------

## 4) Context Managers Preferred

Use:

``` python
with open(...) as f:
    ...
```

Instead of relying on `__del__`.

------------------------------------------------------------------------

## 5) Engineering Takeaways

-   Understand reference semantics
-   Avoid hidden cycles
-   Be cautious with global singletons
-   Manage resources explicitly

------------------------------------------------------------------------
