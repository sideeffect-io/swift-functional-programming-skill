---
name: swift-functional-architecture
description: Functional architecture guidance for Swift 6.2+ with strict concurrency, layering, reducers, state machines, orchestration shells, effect executors, dependency injection, domain modeling, algebraic data types, effects as data, factory-based composition, immutability, and testability.
---

# Functional Architecture in Swift

Use this skill for Swift architecture work where correctness, decoupling, and testability matter more than framework ceremony.

## Core model

Partition the codebase into five concerns:

1. Inert domain
2. Pure decision logic
3. Application orchestration and feature-state management
4. Composition boundary
5. Infrastructure and source-of-truth boundaries

Keep the first two deterministic. Push IO, timers, persistence, networking, and hardware access to the last two.

## Non-negotiables

- Domain types are immutable `struct` and `enum` values by default.
- Reducers and state machines stay pure: `(State, Event) -> Transition<State, Effect>`.
- Feature-state management is a role. If you use a store, keep it a thin orchestration shell.
- Use one named `EffectExecutor` per workflow or use case.
- Executors interpret effects and map capability results into `AsyncStream<Event>`, `Event?`, or `Void`.
- Factories bind immutable feature context, assemble executors, and return ready-to-use feature-state managers.
- Source-of-truth boundaries own authoritative state, merge rules, and cross-source coherence.
- Use `@concurrent` only when leaving caller isolation is intentional.

## Canonical example

The shared teaching example is `EnergyConsumption`, intentionally modeled as a teaching-first Mealy machine with both:

- a stream observation executor
- a one-shot refresh executor

Authoritative split:

- `references/state-machines.md`
  - `Transition`, `EnergyConsumptionState`, `EnergyConsumptionEvent`, `EnergyConsumptionEffect`, exhaustive reducer, transition matrix
- `references/canonical-feature-energy-consumption.md`
  - source of truth, projection, executors, store runtime, factory, view, testing split

## Read by task

| Task | Read |
|---|---|
| Design a new feature end to end | `layers-and-boundaries.md` -> `canonical-feature-energy-consumption.md` -> `new-feature-playbook.md` -> `dependency-injection.md` -> `state-machines.md` -> `testability.md` |
| Refactor a cluttered store | `canonical-feature-energy-consumption.md` -> `orchestration-shells-effect-executors-and-factories.md` -> `state-management-source-of-truth-factory-boundaries.md` -> `dependency-injection.md` |
| Decide ownership between reducer, store, executor, factory, and source of truth | `state-management-source-of-truth-factory-boundaries.md` -> `orchestration-shells-effect-executors-and-factories.md` -> `layers-and-boundaries.md` |
| Decide concurrency semantics | `strict-concurrency-boundaries.md` -> `canonical-feature-energy-consumption.md` |

## Anti-patterns

- Side effects in reducers or domain initializers
- Stores that mix state transitions, wiring, and infrastructure logic
- A large raw `Dependencies` bag as the default feature API
- One executor wrapping multiple unrelated workflows
- Source-of-truth boundaries leaking raw infrastructure payloads upward
- Multiple booleans describing mutually exclusive workflow states

## Optional references

Load only when needed:

- `domain-modeling.md`
- `effects-as-data.md`
- `algebraic-data-types-and-totality.md`
- `function-composition.md`
- `validation-and-error-modeling.md`
- `functional-operators.md`
- `optics.md`
- `solid-in-functional-swift.md`
