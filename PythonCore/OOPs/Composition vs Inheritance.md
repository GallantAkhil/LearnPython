# PythonCore → OOP Deep Dive → Composition vs Inheritance

# 1) Definitions (Practical)

Inheritance: - subclass extends or specializes base behavior - strong
coupling to base class implementation details - tight relationship via
MRO and method overrides

Composition: - object contains other objects and delegates behavior -
looser coupling - more flexible replacement and testing

------------------------------------------------------------------------

# 2) Inheritance: Strengths and Costs

## Strengths

-   code reuse when hierarchy is stable
-   polymorphism via shared interface
-   mixins for small orthogonal behaviors

## Costs

-   fragile base class problem (base changes break subclasses)
-   complex MRO in multiple inheritance
-   hidden coupling and override interactions
-   harder to refactor across boundaries

------------------------------------------------------------------------

# 3) Composition: Strengths and Costs

## Strengths

-   easier testing (swap dependencies)
-   better encapsulation of responsibilities
-   less surprising behavior
-   supports runtime configuration

## Costs

-   more boilerplate delegation
-   need clear interfaces/contracts (Protocols/ABCs)
-   may require more objects and wiring

------------------------------------------------------------------------

# 4) Python-Specific Reality

Because Python is dynamic: - composition + duck typing is extremely
powerful - explicit interfaces can be done with Protocols/ABCs when
needed - inheritance often used for framework integration (Django
models, exceptions, context managers)

Rule of thumb: \> Use inheritance mainly for framework-required base
classes and true "is-a" relationships; use composition for most business
logic.

------------------------------------------------------------------------

# 5) Mixins: A Controlled Use of Inheritance

Mixins work best when: - stateless or minimal state - cooperative
`super()` usage - orthogonal behaviors (logging, serialization helpers)

Avoid mixins that require complex init parameters or heavy state.

------------------------------------------------------------------------

# 6) Backend Patterns

## 6.1 Strategy pattern (composition)

Inject behavior:

``` python
class Service:
    def __init__(self, repo):
        self.repo = repo
```

Test by passing a fake repo.

## 6.2 Adapter pattern (composition)

Wrap third-party clients behind a stable interface.

## 6.3 Domain models

Prefer composition for relationships between entities rather than deep
inheritance chains.

------------------------------------------------------------------------

# 7) Interview Traps

1)  "Prefer composition over inheritance" is a heuristic, not a rule
2)  Inheritance is fine for stable hierarchies and framework contracts
3)  Multiple inheritance requires MRO + cooperative `super()` discipline
4)  Composition improves testability and reduces coupling

------------------------------------------------------------------------
