# Feature State, Source-of-Truth Boundaries, and Factories

These three boundaries drift easily. Keep them explicit.

## Feature-state management

The feature-state layer is a role, not a mandatory `Store` type.

Common shapes:

- a store object acting as a thin orchestration shell over a reducer or state machine
- a directly consumed state machine or `AsyncSequence` of states
- another thin orchestration shell that owns event handling and effect launching

Whatever shape you choose, this layer is an application-orchestration unit.
A store is the common shape when lifecycle hooks, task ownership, or feature-local runtime state matter.

- Hold feature state that changes in response to events
- Apply pure transition logic
- Own task-slot and cancellation policy
- Start and cancel tasks
- Subscribe to observation streams
- Forward named effect-executor outputs back into events

This layer does not:

- Create peer state managers implicitly
- Own authoritative persistent state
- Know how infrastructure is built
- Hide business rules inside task bodies
- Interpret raw capability results inline when named effect executors would keep the shell concise
- Expose a large raw dependency bag as the default public API when explicit executors are clearer

## Observation stream boundaries

Examples in this skill use `AsyncStream` because it is a built-in concrete stream type.

At a real feature or source-of-truth boundary, be explicit about:

- buffering policy
- who finishes the stream
- whether each consumer gets its own stream or many consumers share one

Do not let callers depend on raw `AsyncStream` behavior accidentally.

## Source-of-truth boundaries

Source-of-truth boundaries own authoritative state and cross-source coherence when the architecture needs one.

A repository is one possible implementation shape for such a boundary.
A data source is usually a lower-level provider that the boundary composes.

- Own authoritative snapshots, merges, and write semantics
- Expose domain-focused read and write capabilities
- Compose lower-level storage or client details
- May orchestrate across multiple underlying sources when coherence matters

Source-of-truth boundaries do not:

- Become generic service bags
- Own feature-local transient state
- Return infrastructure-specific payloads upward

## Factories

Factories are composition boundaries.

- Assemble concrete dependencies
- Bind immutable feature context such as identifiers or route parameters
- Build effect executors and feature-state managers with narrow capabilities
- Connect parent and child features
- Choose concrete live, preview, or test wiring
- Prefer returning assembled feature-state managers over intermediate dependency bags when that keeps the composition boundary clearer

Factories do not:

- Hold evolving runtime state
- Contain hidden workflow branches that belong in reducers or source-of-truth boundaries
- Replace source-of-truth boundaries as the place where cross-aggregate invariants live

## Ownership rules

| Concern | Primary owner |
|---|---|
| Pending command state | Feature-state layer |
| Persistent entity state | Source-of-truth boundary |
| Cross-feature wiring | Factory |
| Workflow transition rules | Reducer or state machine |
| Task-slot and cancellation policy | Feature-state layer |
| Effect interpretation | Effect executor |
| Merge and reconciliation rules | Source-of-truth boundary or pure policy |
| Concrete dependency assembly | Factory |

## Example: cross-aggregate orchestration belongs in a source-of-truth boundary

```swift
import Foundation

struct Favorite: Sendable, Equatable {
    let id: UUID
    let title: String
    let rank: Int
}

struct FavoritesSourceOfTruth: Sendable {
    let observe: @Sendable () -> AsyncStream<[Favorite]>
    let reorder: @Sendable ([UUID]) async throws -> Void
}

struct ScenesSourceOfTruth: Sendable {
    let observe: @Sendable () -> AsyncStream<[Favorite]>
    let reorder: @Sendable ([UUID]) async throws -> Void
}

struct DashboardSourceOfTruth: Sendable {
    let observeFavorites: @Sendable () -> AsyncStream<[Favorite]>
    let reorderFavorites: @Sendable ([UUID]) async throws -> Void
}
```

The important idea is not the exact API shape. It is that the cross-source merge and reorder behavior belongs in an explicit source-of-truth boundary, not in ad hoc closure logic inside a feature-state factory. `AsyncStream` is only a concrete example, not a required public contract.

## Coordination rules

- Parent-child feature-state coordination uses explicit closures, events, or composed capabilities.
- If two sources must stay consistent, hide the coordination behind one source-of-truth boundary.
- If multiple feature-state managers need the same authoritative stream, do not duplicate merge logic in every one.
- If a feature-state manager starts owning too many concrete dependencies, move orchestration down into a source-of-truth boundary or up into a factory.

## Smells

- A factory containing reorder or merge algorithms
- A feature-state manager with knowledge of multiple persistence backends
- A source-of-truth boundary returning UI-only flags that should be derived higher up
- A parent feature reaching into child internals instead of sending events or capabilities
