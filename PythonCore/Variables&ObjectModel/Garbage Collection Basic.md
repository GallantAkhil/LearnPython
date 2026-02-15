# PythonCore → Variables & Object Model → Garbage Collection Basics

**Systems-Level Deep Dive (reference counting + cyclic GC +
performance)**

------------------------------------------------------------------------

## 1) Primary Mechanism: Reference Counting

Every object has `ob_refcnt`.

When refcount reaches zero: - object deallocated immediately

------------------------------------------------------------------------

## 2) Cyclic Garbage Collector

Reference counting cannot collect cycles.

Python has generational GC to detect cyclic references.

Generations: - 0 (young) - 1 - 2 (old)

------------------------------------------------------------------------

## 3) Performance Considerations

-   Frequent object creation increases GC activity
-   Large cyclic graphs may delay memory release
-   Use weakref for breaking cycles

------------------------------------------------------------------------

## 4) `__del__` Pitfalls

Objects with `__del__` in cycles may not be collected cleanly.

Prefer context managers over relying on GC finalization.

------------------------------------------------------------------------
