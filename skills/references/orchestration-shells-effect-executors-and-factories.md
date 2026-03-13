# Orchestration Shells, Effect Executors, and Factories

Use this split when a feature has pure workflow logic plus async observation, commands, or timers.
The running example is `EnergyConsumption`.

## Default layering

1. Domain values
2. Pure reducer or state machine
3. Orchestration shell or store
4. Factory or composition boundary
5. Infrastructure and source-of-truth boundaries

The goal is simple:

- reducers decide
- stores orchestrate
- executors interpret
- factories assemble
- source-of-truth boundaries own authoritative data

## Store responsibilities

The store is a thin runtime shell around pure transition logic.

It should:

- own local runtime state and task slots
- dispatch events into the reducer or state machine
- start and cancel tracked tasks
- forward executor outputs back into the workflow as events
- keep lifecycle and cancellation policy explicit

It should not:

- expose a large raw `Dependencies` bag as the default public API
- create peer stores implicitly
- assemble infrastructure
- hide business rules inside task bodies

## Executor responsibilities

An effect executor interprets one workflow only.

| Workflow kind | Shape |
|---|---|
| Observation | `async -> AsyncStream<Event>` |
| One-shot workflow | `async -> Event?` |
| Fire-and-forget command | `async -> Void` |

Default rules:

- one executor per workflow or use case
- concrete `Sendable` struct
- `callAsFunction`
- narrow closures or capability structs
- `@concurrent` only when leaving caller isolation is intentional

## Factory responsibilities

Factories are the composition boundary.

- bind immutable context such as identifiers or route parameters
- build executors from repositories, clients, runtimes, and clocks
- return the assembled feature-state manager
- own live, preview, and test wiring

## Canonical runtime skeleton

Using the canonical `EnergyConsumption` types:

```swift
@MainActor
final class EnergyConsumptionStore {
    private let observeEnergyConsumption: ObserveEnergyConsumptionEffectExecutor
    private let refreshEnergyConsumption: RefreshEnergyConsumptionEffectExecutor
    private var observationTask: Task<Void, Never>?
    private var refreshTask: Task<Void, Never>?

    func send(_ event: EnergyConsumptionEvent) { /* reducer entrypoint */ }
    private func handle(_ effect: EnergyConsumptionEffect) { /* runtime dispatch */ }
}

struct EnergyConsumptionStoreFactory: Sendable {
    let deviceRepository: DeviceRepository

    @MainActor
    func make(identifier: DeviceIdentifier) -> EnergyConsumptionStore { /* bind and assemble */ }
}
```

The full runtime wiring lives in `canonical-feature-energy-consumption.md`.

## Ownership rules

| Concern | Primary owner |
|---|---|
| Transition logic | Reducer or state machine |
| Task-slot and cancellation policy | Orchestration shell or store |
| Effect interpretation | Effect executor |
| Concrete assembly and immutable context binding | Factory |
| Authoritative persistence and coherence | Source-of-truth boundary |

## Testing split

- reducer tests prove workflow transitions
- executor tests prove workflow-to-event mapping
- store tests prove task ownership and event forwarding
- factory tests stay light and verify wiring that is easy to break

## Smells

- a store initializer that takes a large `Dependencies` bag
- one executor wrapping multiple unrelated workflows
- a factory returning a dependency bag instead of an assembled feature
- a reducer encoding runtime policy
