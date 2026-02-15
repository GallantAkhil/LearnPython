# PythonCore → OOP Deep Dive → Class Structure (CPython Object Model)

# 1) Conceptual Model: Class as a Runtime Object

In Python, a class definition executes and produces a class object:

``` python
class User:
    role = "member"
    def greet(self): ...
```

At runtime:

-   `User` is an object
-   `type(User) is type` (unless custom metaclass)
-   `User.__dict__` is a mapping of class attributes
-   Instances have `obj.__class__ is User`

------------------------------------------------------------------------

# 2) CPython Core Layout: `PyObject` Header

Every object begins with:

-   refcount
-   type pointer (`ob_type`)

So for an instance `u = User()`:

-   `u` has pointer to `User` (its type)
-   `User` has pointer to `type` (its metatype)

This is why Python can do runtime dispatch and introspection cheaply.

------------------------------------------------------------------------

# 3) Class Creation Pipeline

## 3.1 What `class` statement does (high level)

1)  Execute class body in a temporary namespace dict
2)  Call metaclass `__prepare__` (optional) to create the namespace
    mapping
3)  Execute body → fills namespace dict
4)  Call metaclass `__new__` to create class object
5)  Call metaclass `__init__` to finalize class
6)  Bind resulting class object to the class name

Key point: \> The class body is executed like normal code; decorators
and executed expressions run at definition time.

## 3.2 `type()` is the default metaclass

The default metaclass is `type`. You can see it as:

``` python
User = type("User", (object,), {"role": "member", "greet": greet_fn})
```

Metaclasses allow altering step 4/5 (class object creation).

------------------------------------------------------------------------

# 4) Instance Structure: Where Attributes Live

By default, instances have an attribute dictionary:

``` python
u.__dict__  # {'name': 'Akhil', ...}
```

Internally, instance attribute storage is typically:

-   `__dict__` pointer to a hash table (PyDictObject)
-   Optional `__weakref__` slot
-   Plus the base object header

This is flexible but has overhead.

------------------------------------------------------------------------

# 5) Attribute Access Algorithm (Critical)

When you do:

``` python
u.attr
```

Python does *not* simply look inside `u.__dict__`. The resolution
algorithm is roughly:

1)  Look for **data descriptors** on the class (and bases via MRO)
    -   data descriptor = defines `__set__` or `__delete__` (e.g.,
        `property`)
2)  Look in `u.__dict__` (instance attributes)
3)  Look for non-data descriptors / plain attributes on class via MRO
    -   functions are non-data descriptors, bind into methods here
4)  If found descriptor, invoke `__get__`
5)  If not found, call `__getattr__` (if defined)
6)  Otherwise raise `AttributeError`

This is why: - properties override instance attributes - functions
become bound methods - `__getattribute__` can intercept everything
(dangerous)

------------------------------------------------------------------------

# 6) Descriptors: The Engine Behind Methods & Properties

A descriptor is any object with any of: -
`__get__(self, obj, objtype)` - `__set__(self, obj, value)` -
`__delete__(self, obj)`

## 6.1 Functions are descriptors

A function defined in a class has `__get__`. When accessed via instance,
it binds `self`:

``` python
u.greet   # bound method (wraps function + u)
User.greet # function (unbound)
```

## 6.2 `property` is a data descriptor

Because it defines `__set__` (or at least participates as data
descriptor), it takes precedence over instance dict.

This can be used for invariants, computed values, and lazy loading ---
but can be expensive in hot paths.

------------------------------------------------------------------------

# 7) `__slots__`: Changing Instance Layout

`__slots__` removes per-instance `__dict__` (unless explicitly included)
and allocates a fixed layout for attributes.

``` python
class User:
    __slots__ = ("id", "name")
```

Effects: - Less memory per instance - Faster attribute access (no dict
hash) - But less flexible: you can't add arbitrary attributes

Engineering trade-off: - great for large numbers of small objects
(ORM-like DTOs, AST nodes) - can complicate multiple inheritance and
weakref support

------------------------------------------------------------------------

# 8) Method Resolution Order (Preview)

Class attribute lookup uses MRO (C3 linearization). This matters heavily
in multiple inheritance (covered in its own file).

------------------------------------------------------------------------

# 9) Production Engineering Notes

-   Avoid overriding `__getattribute__` unless you can prove correctness
-   Use `property` for invariants but be aware of attribute access cost
-   Consider `__slots__` for memory-critical data models
-   Preserve introspection friendliness (especially in frameworks:
    FastAPI/Pydantic)

------------------------------------------------------------------------

# 10) Interview Traps

1)  A class is an object (instance of `type`)
2)  `u.attr` lookup uses descriptor protocol + MRO, not just `__dict__`
3)  Functions become bound methods via descriptors
4)  `property` can override instance attributes
5)  `__slots__` changes memory layout and attribute behavior

------------------------------------------------------------------------
