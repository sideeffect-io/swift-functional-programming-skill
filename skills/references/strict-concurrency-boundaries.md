# Strict Concurrency Boundaries

This skill assumes Swift 6.2+ architecture and stays explicit about execution semantics.
The running example is `EnergyConsumption`.

## Baseline assumptions

- `Sendable` is part of boundary design
- `@concurrent` is a deliberate boundary choice, not a default
- actors are for true isolated mutable state, not for wrapping stateless helpers

## The key Swift 6.2 distinction

When `NonisolatedNonsendingByDefault` is enabled:

- plain nonisolated async functions inherit the caller's actor
- `@concurrent` functions leave caller isolation and run on the generic executor

Use that distinction intentionally.

## Default choices

| Need | Use |
|---|---|
| Pure logic | Plain function |
| Pure helper inside isolated type | `nonisolated` helper |
| Async helper that should inherit caller isolation | Plain async or `nonisolated(nonsending)` |
| Async work that must intentionally leave caller isolation | `@concurrent` |
| Stateful serialized boundary | Actor |

## Canonical executor signatures

Using the `EnergyConsumption` example:

```swift
struct ObserveEnergyConsumptionEffectExecutor: Sendable {
    @concurrent
    func callAsFunction() async -> AsyncStream<EnergyConsumptionEvent>
}

struct RefreshEnergyConsumptionEffectExecutor: Sendable {
    @concurrent
    func callAsFunction() async -> EnergyConsumptionEvent?
}
```

The canonical feature reference shows the full executor bodies. The important point here is the contract: one executor intentionally leaves caller isolation for a stream workflow, and one does the same for a one-shot workflow.

## Keep pure helpers off the actor

```swift
@MainActor
final class EnergyConsumptionStore {
    struct DeviceSnapshot: Sendable, Equatable {
        let kilowattHours: Double?
        let unitSymbol: String
    }

    nonisolated static func makeDescriptor(
        from snapshot: DeviceSnapshot
    ) -> EnergyConsumptionDescriptor? { /* pure helper */ }
}
```

If no isolated mutable state is involved, do not let actor isolation leak into pure helpers by accident.

## Smells

- a worker actor with no meaningful protected state
- `@MainActor` used to silence sendability issues instead of modeling isolation
- async helpers whose execution semantics are unclear from the signature
- `@unchecked Sendable` used where better boundaries would remove the need
