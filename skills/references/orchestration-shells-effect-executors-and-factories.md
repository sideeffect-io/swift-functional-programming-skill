# Orchestration Shells, Effect Executors, and Factories

Use this split when a feature has pure workflow logic plus async observation, commands, or timers.

## Default feature layering

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

## Orchestration shell or store

The store is a thin runtime shell around pure transition logic.

It should:

- own local runtime state and task slots
- dispatch events into the reducer or state machine
- start and cancel tracked tasks
- forward executor outputs back into the workflow as events
- keep lifecycle and cancellation policy explicit
- stay concise

It should not:

- expose a large raw `Dependencies` bag as the default public API
- create peer stores implicitly
- assemble infrastructure
- hide business rules inside task bodies
- interpret raw capability results inline when a named executor would keep the shell simple

## Effect executors

An effect executor interprets one workflow only.

Default rules:

- use one executor per workflow or observation
- prefer a concrete `Sendable` struct, not a protocol
- expose `callAsFunction`
- depend on narrow closures or capability structs
- translate raw capability results into feature events or fire-and-forget completion
- add `@concurrent` only when the executor must intentionally leave caller isolation

Default shapes:

| Workflow kind | Default shape |
|---|---|
| Observation | `func callAsFunction(...) async -> AsyncStream<Event>` |
| One-shot effect that feeds the reducer | `func callAsFunction(...) async -> Event?` |
| Fire-and-forget command | `func callAsFunction(...) async -> Void` |

Example:

```swift
import Foundation

struct ObserveDevicesEffectExecutor: Sendable {
    let observe: @Sendable () -> AsyncStream<DevicesEvent>

    @concurrent
    func callAsFunction() async -> AsyncStream<DevicesEvent> {
        observe()
    }
}

struct RefreshDevicesEffectExecutor: Sendable {
    let refresh: @Sendable () async throws -> [Device]

    @concurrent
    func callAsFunction() async -> DevicesEvent? {
        do {
            let devices = try await refresh()
            return .devicesWereRefreshed(devices)
        } catch {
            return .refreshDidFail(message: String(describing: error))
        }
    }
}

struct ToggleFavoriteEffectExecutor: Sendable {
    let toggleFavorite: @Sendable () async -> Void

    @concurrent
    func callAsFunction() async {
        await toggleFavorite()
    }
}
```

## Factories

Factories are the composition boundary for a feature.

They should:

- bind immutable context such as identifiers or feature parameters
- build effect executors from repositories, clients, runtimes, and clocks
- construct and return the assembled store
- own live, preview, and test wiring

They should prefer returning ready-to-use feature-state managers over intermediate dependency bags when that keeps the boundary clearer.

## Ownership rules

| Concern | Primary owner |
|---|---|
| Transition logic | Reducer or state machine |
| Task-slot and cancellation policy | Orchestration shell or store |
| Effect interpretation | Effect executor |
| Concrete assembly and immutable context binding | Factory |
| Authoritative persistence, merge rules, and coherence | Source-of-truth boundary |

## Smells

- a store initializer that takes a large `Dependencies` bag
- one executor that wraps multiple unrelated workflows
- a factory that returns a dependency bag instead of an assembled feature
- a reducer that encodes cancellation or runtime policy
- a store that mixes state transitions, wiring, and infrastructure logic

## Testing split

- reducer tests prove workflow transitions
- executor tests prove workflow-to-event or workflow-to-result mapping
- store tests prove task ownership, cancellation, and event forwarding
- factory tests stay light and only verify wiring that is otherwise easy to break
