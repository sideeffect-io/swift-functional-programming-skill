# Domain Modeling

Model the problem first. Architecture quality is downstream from type quality.
The running example is `EnergyConsumption`.

## Defaults

- `struct` for product types
- `enum` for alternatives, workflows, and constrained modes
- explicit value objects for values with invariants

## Encode invariants in types

```swift
struct Percentage: Sendable, Equatable {
    let value: Int

    init?(_ value: Int) {
        guard (0...100).contains(value) else { return nil }
        self.value = value
    }
}
```

Reject invalid values at the edge instead of patching them later.

## Prefer workflow states over boolean piles

Using the canonical `EnergyConsumptionState` from `state-machines.md`:

- `idle`
- `awaitingFirstSample`
- `showing(EnergyConsumptionDescriptor)`
- `refreshing(EnergyConsumptionDescriptor?)`
- `unavailable`

This is clearer than `hasStarted`, `isLoading`, `isRefreshing`, `descriptor`, and `isUnavailable`.
The `refreshing(EnergyConsumptionDescriptor?)` state also makes “last known value plus in-flight work” explicit without parallel booleans.

## Separate authoritative and derived state

In the canonical feature:

- `Device` is authoritative
- `EnergyConsumptionDescriptor` is derived
- `EnergyConsumptionProjection` performs the derivation

Do not persist or hand-edit derived values as if they were a second source of truth.

## Naming rules

- nouns for domain values
- past-tense for observed facts
- imperative names only at effect boundaries
- workflow state names explicit enough that illegal combinations are obvious

## Smells

- multiple optionals whose presence depends on a hidden mode
- a “kind” field plus many unrelated stored properties
- domain values that know transport, storage, or framework details
