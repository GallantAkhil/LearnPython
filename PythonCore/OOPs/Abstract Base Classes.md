# PythonCore → OOP Deep Dive → Abstract Base Classes (ABCs)

# 1) What an ABC Is

An Abstract Base Class is a class that: - cannot be instantiated until
abstract methods are implemented - defines a contract for subclasses

Basic:

``` python
from abc import ABC, abstractmethod

class Repo(ABC):
    @abstractmethod
    def get(self, id): ...
```

If a subclass doesn't implement `get`, instantiation fails with
`TypeError`.

------------------------------------------------------------------------

# 2) CPython Mechanics: `ABCMeta`

ABCs use metaclass `ABCMeta` which: - tracks abstract methods set -
computes `__abstractmethods__` - blocks instantiation if abstract
methods remain

Enforcement happens at runtime when calling the class (during
`type.__call__`).

------------------------------------------------------------------------

# 3) Abstract Methods Variants

You can combine `@abstractmethod` with:

-   `@property`
-   `@classmethod`
-   `@staticmethod`

Pattern examples:

``` python
class X(ABC):
    @property
    @abstractmethod
    def name(self): ...

    @classmethod
    @abstractmethod
    def from_row(cls, row): ...
```

Important: decorator order matters (`@property` then `@abstractmethod`
or vice versa depending on style), but the end result must mark the
attribute as abstract.

------------------------------------------------------------------------

# 4) Virtual Subclassing

You can register a class as a "virtual subclass" without inheritance:

``` python
Repo.register(MyRepoImplementation)
```

Then: - `issubclass(MyRepoImplementation, Repo)` becomes True - but MRO
is unchanged (no method injection)

This is useful for plugin systems but can confuse maintainers if abused.

------------------------------------------------------------------------

# 5) ABCs vs Protocols

-   ABCs: runtime enforcement + `isinstance` checks + possible default
    implementations
-   Protocols (typing): structural typing for static analysis; runtime
    checks optional with `runtime_checkable`

In backend engineering: - use Protocols for flexibility and testing -
use ABCs when runtime enforcement matters or you ship a plugin API and
want clear instantiation-time failures

------------------------------------------------------------------------

# 6) Production Patterns

## 6.1 Plugin architectures

ABCs define the contract; implementations loaded dynamically.

## 6.2 Adapter layers (ports & adapters)

ABCs or Protocols define ports; adapters implement them.

## 6.3 Default implementations in ABCs

Provide shared helper methods to reduce duplication.

------------------------------------------------------------------------

# 7) Pitfalls

-   Don't over-abstract: too many ABCs create rigidity
-   Virtual subclassing can make type relationships non-obvious
-   ABC checks can add overhead in hot paths if overused (usually
    negligible)

------------------------------------------------------------------------

# 8) Interview Traps

1)  ABCs use metaclass `ABCMeta` and enforce at instantiation time
2)  `@abstractmethod` prevents instantiation until implemented
3)  `register` makes virtual subclasses without inheritance
4)  Protocols differ: structural typing, mostly static

------------------------------------------------------------------------
