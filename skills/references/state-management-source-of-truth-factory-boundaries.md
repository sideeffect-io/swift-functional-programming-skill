# Feature State, Source-of-Truth Boundaries, and Factories

These three boundaries drift easily. Keep them explicit.
The running example is `EnergyConsumption`.

## Feature-state layer

This is a role, not a mandatory `Store` type.

It owns:

- feature-local workflow state
- pure transition application
- task-slot and cancellation policy
- subscription and command launching
- forwarding executor output back into events

It does not own:

- authoritative persistent state
- infrastructure assembly
- cross-source merge rules

## Source-of-truth boundary

A source-of-truth boundary owns authoritative state and coherence.

It owns:

- authoritative snapshots
- observation and write capabilities
- merge and reconciliation rules
- cross-source coordination when coherence matters

It does not own:

- feature-local transient workflow state
- UI-only derived flags

## Factory

The factory is the composition boundary.

It owns:

- concrete dependency assembly
- immutable context binding
- executor construction
- parent/child feature wiring

It does not own:

- evolving runtime state
- reducer logic
- source-of-truth merge algorithms

## Ownership map for `EnergyConsumption`

| Concern | Owner |
|---|---|
| Authoritative device state | `DeviceRepository` |
| Derived descriptor | `EnergyConsumptionProjection` |
| Workflow state | `EnergyConsumptionState` |
| Task slots | `EnergyConsumptionStore` |
| Effect interpretation | `ObserveEnergyConsumptionEffectExecutor`, `RefreshEnergyConsumptionEffectExecutor` |
| Identifier binding and assembly | `EnergyConsumptionStoreFactory` |

## If multiple sources must stay coherent

Move that coordination into a source-of-truth boundary, not a factory or store.

```swift
struct DashboardSourceOfTruth: Sendable {
    let observeFavorites: @Sendable () -> AsyncStream<[Favorite]>
    let reorderFavorites: @Sendable ([UUID]) async throws -> Void
}
```

The important point is the ownership, not the exact API shape: cross-source merge and reorder logic belongs in one authoritative boundary.

## Smells

- a factory containing reorder or merge algorithms
- a feature-state manager with knowledge of multiple persistence backends
- a source-of-truth boundary returning UI-only flags that should be derived higher up
- a parent feature reaching into child internals instead of sending events or capabilities
