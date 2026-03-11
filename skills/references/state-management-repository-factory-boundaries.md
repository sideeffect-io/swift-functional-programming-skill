# Feature State, Repository, and Factory Boundaries

These three boundaries drift easily. Keep them explicit.

## Feature-state management

The feature-state layer is a role, not a mandatory `Store` type.

Common shapes:

- a store object that exposes feature state and intents
- a directly consumed state machine or `AsyncSequence` of states
- another thin orchestration shell that owns event handling and effect launching

Whatever shape you choose, this layer is an application-orchestration unit.

- Hold feature state that changes in response to events
- Apply pure transition logic
- Start and cancel tasks
- Subscribe to observation streams
- Map effect results back into events

This layer does not:

- Create peer state managers implicitly
- Own authoritative persistent state
- Know how infrastructure is built
- Hide business rules inside task bodies

## Observation stream boundaries

Examples in this skill use `AsyncStream` because it is a built-in concrete stream type.

At a real feature or repository boundary, be explicit about:

- buffering policy
- who finishes the stream
- whether each consumer gets its own stream or many consumers share one

Do not let callers depend on raw `AsyncStream` behavior accidentally.

## Repositories

Repositories are source-of-truth boundaries.

- Own authoritative snapshots, merges, and write semantics
- Expose domain-focused read and write capabilities
- Compose lower-level storage or client details
- May orchestrate across multiple underlying sources when coherence matters

Repositories do not:

- Become generic service bags
- Own feature-local transient state
- Return infrastructure-specific payloads upward

## Factories

Factories are composition boundaries.

- Assemble concrete dependencies
- Build feature-state managers with narrow capabilities
- Connect parent and child features
- Choose concrete live, preview, or test wiring

Factories do not:

- Hold evolving runtime state
- Contain hidden workflow branches that belong in reducers or repositories
- Replace repositories as the place where cross-aggregate invariants live

## Ownership rules

| Concern | Primary owner |
|---|---|
| Pending command state | Feature-state layer |
| Persistent entity state | Repository |
| Cross-feature wiring | Factory |
| Workflow transition rules | Reducer or state machine |
| Merge and reconciliation rules | Repository or pure policy |
| Concrete dependency assembly | Factory |

## Example: cross-aggregate orchestration belongs in a repository

```swift
import Foundation

struct Favorite: Sendable, Equatable {
    let id: UUID
    let title: String
    let rank: Int
}

struct FavoritesRepository: Sendable {
    let observe: @Sendable () -> AsyncStream<[Favorite]>
    let reorder: @Sendable ([UUID]) async throws -> Void
}

struct ScenesRepository: Sendable {
    let observe: @Sendable () -> AsyncStream<[Favorite]>
    let reorder: @Sendable ([UUID]) async throws -> Void
}

struct DashboardRepository: Sendable {
    let observeFavorites: @Sendable () -> AsyncStream<[Favorite]>
    let reorderFavorites: @Sendable ([UUID]) async throws -> Void
}
```

The important idea is not the exact API shape. It is that the cross-source merge and reorder behavior belongs in an explicit repository boundary, not in ad hoc closure logic inside a feature-state factory. `AsyncStream` is only a concrete example, not a required public contract.

## Coordination rules

- Parent-child feature-state coordination uses explicit closures, events, or composed capabilities.
- If two sources must stay consistent, hide the coordination behind one repository boundary.
- If multiple feature-state managers need the same authoritative stream, do not duplicate merge logic in every one.
- If a feature-state manager starts owning too many concrete dependencies, move orchestration down into a repository or up into a factory.

## Smells

- A factory containing reorder or merge algorithms
- A feature-state manager with knowledge of multiple persistence backends
- A repository returning UI-only flags that should be derived higher up
- A parent feature reaching into child internals instead of sending events or capabilities
