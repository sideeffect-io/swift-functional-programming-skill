# Dependency Injection

Prefer the smallest boundary that preserves clarity.

## Order of preference

1. Closures
2. Capability structs
3. Protocols at external boundaries
4. Actors only when stateful isolation is inherent

## Closures first

Use direct closures when a unit needs one or two operations.

```swift
enum SavePersonUseCase {
    static func make(
        save: @escaping @Sendable (Person) async throws -> Void,
        reportFailure: @escaping @Sendable (Person, SavePersonFailure) async -> Void
    ) -> @Sendable (Person) async -> Void
}
```

Use capability structs when several operations belong to one boundary or when named overrides improve test ergonomics.

## Effectful feature seams

For reducer-driven features, the default seam is a named `EffectExecutor`.

- one executor per workflow or use case
- concrete `Sendable` struct over a protocol by default
- `callAsFunction`
- narrow closures or capability structs as dependencies
- `AsyncStream<Event>` for observation, `Event?` for one-shot workflows, `Void` for fire-and-forget commands

The canonical example is `EnergyConsumption`.

## Factory binding in the canonical feature

Using the types defined in `canonical-feature-energy-consumption.md`:

```swift
struct EnergyConsumptionStoreFactory: Sendable {
    let deviceRepository: DeviceRepository

    @MainActor
    func make(identifier: DeviceIdentifier) -> EnergyConsumptionStore {
        EnergyConsumptionStore(
            observeEnergyConsumption: .init(
                observeEnergyConsumption: {
                    await deviceRepository.observeByID(identifier)
                }
            ),
            refreshEnergyConsumption: .init(
                refreshEnergyConsumption: {
                    await deviceRepository.refreshByID(identifier)
                }
            )
        )
    }
}
```

This is the important split:

- the store does not receive a large `Dependencies` bag
- the factory binds immutable context
- the factory assembles concrete executors
- the store receives ready-to-run collaborators

## Protocols are boundary tools

Prefer protocols only when you need:

- runtime polymorphism across modules
- framework integration that expects a protocol/reference boundary
- a public extension point
- a generic constraint

If the core does not need polymorphism, adapt the protocol to closures or capability structs at the boundary.

## Concurrency rules

- dependency closures that may cross isolation boundaries should be `@Sendable`
- use `@concurrent` on executors only when leaving caller isolation is part of the intended semantics
- do not add actors just to make async seams look explicit

## Smells

- one giant environment object shared by unrelated features
- protocols created only to make mocking possible
- dependencies passed through layers untouched for no reason
- hidden globals instead of explicit factory wiring
