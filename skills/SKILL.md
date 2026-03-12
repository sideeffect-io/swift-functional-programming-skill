---
name: swift-functional-architecture
description: Functional architecture guidance for Swift 6.2+ with strict concurrency, layering, reducers, state machines, orchestration shells, effect executors, dependency injection, domain modeling, algebraic data types, effects as data, factory-based composition, immutability, and testability.
---

# Functional Architecture in Swift

Use this skill for Swift architecture work where correctness, decoupling, and testability matter more than framework ceremony.

## When to use

- Domain modeling and workflow design
- Reducers, state machines, and orchestration layers
- Layering, boundaries, and dependency direction
- Dependency injection and composition roots
- Strict concurrency architecture in Swift 6.2+
- Refactors that should improve testability without adding abstraction noise

## North star

Partition the codebase into five concerns:

1. Inert domain
2. Pure decision logic
3. Application orchestration and feature-state management
4. Composition boundary
5. Infrastructure

Keep the first two fully deterministic. Push side effects, storage, networking, timers, and hardware access to the last two.

## Functional orientation

This is a functional-programming-oriented architecture skill, not a generic layered-architecture guide.

- Prefer immutable values, algebraic data types, exhaustive pattern matching, and pure transformations.
- Keep decision logic referentially transparent whenever possible.
- Describe effects as data and interpret them at the boundary.
- Borrow heavily from pure-functional architecture style, while staying idiomatic in Swift's type system and concurrency model.

## Layer and communication rules

- Domain types are immutable `struct` and `enum` values by default.
- Pure decision logic is synchronous whenever possible and returns data, not side effects.
- Reducers remain pure: `(State, Event) -> Transition<State, Effect>`.
- Feature-state management is a role, not a mandatory type. It may be a store, a directly consumed state machine, or another thin orchestration shell.
- If you use stores, keep them thin orchestration shells. They own runtime state, task slots, lifecycle hooks, and event forwarding, but not large dependency bags or infrastructure assembly.
- Prefer one named `EffectExecutor` per workflow. Executors interpret emitted effects and map capability results into `AsyncStream<Event>`, `Event?`, or `Void`.
- Source-of-truth boundaries own authoritative state and cross-aggregate orchestration when a source of truth must stay coherent.
- Factories and bootstrap code own wiring. They bind immutable feature context, assemble executors, and return ready-to-use feature-state managers.
- Inward layers depend on capabilities, not concrete implementations.
- Prefer composition over inheritance. Inheritance is a framework constraint, not an architecture default.

## Strict concurrency defaults

- Assume Swift 6.2+ with strict concurrency in mind.
- Mark boundary values and dependency closures `Sendable` when that is semantically correct.
- Use `@concurrent` sparingly, only for effect executors that must intentionally leave caller isolation and run concurrently with the caller.
- If an async helper should stay on the caller's actor, rely on `NonisolatedNonsendingByDefault` when enabled; otherwise spell `nonisolated(nonsending)` explicitly in reusable examples.
- Prefer pure or `nonisolated` helpers over actor-isolated helpers when no isolated mutable state is needed.
- Use actors for true serialization boundaries only: shared mutable caches, authoritative source-of-truth boundaries, or coordination points with identity and lifecycle.

## Decision table

| Need | Default choice |
|---|---|
| Pure business rule | Free or namespaced pure function |
| Pure state transition | Reducer or state machine transition |
| One async effect that must intentionally run off the caller actor | Named `@concurrent` `EffectExecutor` returning a stream, event, or result |
| A few dependencies | Individual closure parameters |
| A related group of dependencies | `Sendable` capability struct |
| Runtime polymorphism or external integration seam | Protocol at the outer boundary |
| Shared mutable state | Actor or source-of-truth boundary, not a default worker actor |

## Anti-patterns

- Side effects in reducers or domain value initializers
- Hidden feature-state managers created inside other feature-state managers
- Parallel APIs for the "fast path" and the "real path" instead of modeling both inside one orchestrator
- Effect builders that hide lifecycle or cancellation policy outside the workflow model
- Protocol per concrete type when a closure or capability struct would do
- An actor per feature by default
- Source-of-truth boundaries that leak raw infrastructure details upward
- Multiple booleans that describe mutually exclusive workflow states
- Stores that mix state transitions, wiring, and infrastructure logic

## Reference map

Load only the files needed for the task.

- `references/layers-and-boundaries.md`
  - Start here for the architecture map and dependency direction rules.
- `references/new-feature-playbook.md`
  - Read when implementing a feature end-to-end and you need a whole-feature workflow, not just isolated architecture rules.
- `references/state-management-source-of-truth-factory-boundaries.md`
  - Read when responsibilities are drifting between feature-state management, source-of-truth ownership, and composition.
- `references/orchestration-shells-effect-executors-and-factories.md`
  - Read when deciding what belongs in a reducer, store, effect executor, factory, or source-of-truth boundary.
- `references/solid-in-functional-swift.md`
  - Read when discussing SOLID, composition over inheritance, and abstraction quality.
- `references/domain-modeling.md`
  - Read for immutable modeling, ADTs, invariants, derived state, and source-of-truth rules.
- `references/dependency-injection.md`
  - Read for closures-first DI, capability structs, protocol boundaries, and composition-root assembly.
- `references/strict-concurrency-boundaries.md`
  - Read for `Sendable`, `@concurrent`, `nonisolated`, and Swift 6.2 execution semantics.
- `references/state-machines.md`
  - Read for reducer-driven workflows, effect execution, lifecycle modeling, and parent-child coordination.
- `references/testability.md`
  - Read for testing seams by layer and how strict concurrency affects test design.
- For broader FP grounding that still applies directly to Swift architecture:
- `references/algebraic-data-types-and-totality.md`
  - Optional. Read for product and sum types, illegal-state elimination, exhaustive matching, and total-function thinking.
- `references/function-composition.md`
  - Optional. Read for higher-order functions, composition, partial application, and when to extract pure helpers.
- `references/effects-as-data.md`
  - Optional. Read for describing side effects as values plus interpreters at the boundary.
- `references/validation-and-error-modeling.md`
  - Optional. Read for `Optional` and `Result` pipelines, fail-fast validation, and domain errors.
- `references/functional-operators.md`
  - Optional. Read for lightweight operator guidance in pure transformations.
- `references/optics.md`
  - Optional. Read when immutable nested updates are getting noisy.

## Practical heuristics

- Prefer enums as namespaces for related pure functions when a type carries no state.
- Keep functions small enough that their contract is obvious without scrolling.
- Keep files cohesive; split by responsibility rather than by arbitrary suffixes.
- If a unit is hard to test, the design probably mixes boundaries that should be separate.
