# State Machines

Use a state machine when the workflow, not the view hierarchy, is the hard part.
The running example is `EnergyConsumption`.

This file is authoritative for `Transition`, `EnergyConsumptionState`, `EnergyConsumptionEvent`, `EnergyConsumptionEffect`, the exhaustive reducer, and the transition matrix.
`EnergyConsumptionDescriptor`, `EnergyConsumptionProjection`, executors, store runtime, factory, and view live in `canonical-feature-energy-consumption.md`.

## When to use one

Use a state machine when:

- transitions depend on both mode and context
- effects feed results back into the workflow
- lifecycle, cancellation, retries, or parallel flows must be explicit
- you need to prove where orchestration logic lives

## Core shape

```swift
struct Transition<State: Sendable, Effect: Sendable>: Sendable {
    let state: State
    let effects: [Effect]
}
```

The core contract is:

```swift
(State, Event) -> Transition<State, Effect>
```

## Authoritative example: `EnergyConsumption`

Using `EnergyConsumptionDescriptor` from the canonical feature reference:

```swift
enum EnergyConsumptionState: Sendable, Equatable {
    case idle
    case awaitingFirstSample
    case showing(EnergyConsumptionDescriptor)
    case refreshing(EnergyConsumptionDescriptor?)
    case unavailable
}

enum EnergyConsumptionEvent: Sendable, Equatable {
    case onAppear
    case refreshWasRequested
    case descriptorWasReceived(EnergyConsumptionDescriptor?)
}

enum EnergyConsumptionEffect: Sendable, Equatable {
    case startObserving
    case refresh
}

enum EnergyConsumptionStateMachine {
    static func reduce(
        _ state: EnergyConsumptionState,
        _ event: EnergyConsumptionEvent
    ) -> Transition<EnergyConsumptionState, EnergyConsumptionEffect> {
        switch (state, event) {
        case (.idle, .onAppear):
            return .init(state: .awaitingFirstSample, effects: [.startObserving])

        case (.idle, .refreshWasRequested):
            return .init(state: .refreshing(nil), effects: [.refresh])

        case (.idle, .descriptorWasReceived(_)):
            return .init(state: .idle, effects: [])

        case (.awaitingFirstSample, .onAppear):
            return .init(state: .awaitingFirstSample, effects: [])

        case (.awaitingFirstSample, .refreshWasRequested):
            return .init(state: .awaitingFirstSample, effects: [])

        case (.awaitingFirstSample, .descriptorWasReceived(nil)):
            return .init(state: .unavailable, effects: [])

        case (.awaitingFirstSample, .descriptorWasReceived(.some(let descriptor))):
            return .init(state: .showing(descriptor), effects: [])

        case (.showing(let current), .onAppear):
            return .init(state: .showing(current), effects: [])

        case (.showing(let current), .refreshWasRequested):
            return .init(state: .refreshing(current), effects: [.refresh])

        case (.showing(_), .descriptorWasReceived(nil)):
            return .init(state: .unavailable, effects: [])

        case (.showing(_), .descriptorWasReceived(.some(let descriptor))):
            return .init(state: .showing(descriptor), effects: [])

        case (.refreshing(let current), .onAppear):
            return .init(state: .refreshing(current), effects: [])

        case (.refreshing(let current), .refreshWasRequested):
            return .init(state: .refreshing(current), effects: [])

        case (.refreshing(_), .descriptorWasReceived(nil)):
            return .init(state: .unavailable, effects: [])

        case (.refreshing(_), .descriptorWasReceived(.some(let descriptor))):
            return .init(state: .showing(descriptor), effects: [])

        case (.unavailable, .onAppear):
            return .init(state: .unavailable, effects: [])

        case (.unavailable, .refreshWasRequested):
            return .init(state: .refreshing(nil), effects: [.refresh])

        case (.unavailable, .descriptorWasReceived(nil)):
            return .init(state: .unavailable, effects: [])

        case (.unavailable, .descriptorWasReceived(.some(let descriptor))):
            return .init(state: .showing(descriptor), effects: [])
        }
    }
}
```

## Transition matrix

| Current state | Event | Next state | Effects |
|---|---|---|---|
| `idle` | `onAppear` | `awaitingFirstSample` | `[.startObserving]` |
| `idle` | `refreshWasRequested` | `refreshing(nil)` | `[.refresh]` |
| `idle` | `descriptorWasReceived(_)` | `idle` | `[]` |
| `awaitingFirstSample` | `onAppear` | `awaitingFirstSample` | `[]` |
| `awaitingFirstSample` | `refreshWasRequested` | `awaitingFirstSample` | `[]` |
| `awaitingFirstSample` | `descriptorWasReceived(nil)` | `unavailable` | `[]` |
| `awaitingFirstSample` | `descriptorWasReceived(.some(descriptor))` | `showing(descriptor)` | `[]` |
| `showing(current)` | `onAppear` | `showing(current)` | `[]` |
| `showing(current)` | `refreshWasRequested` | `refreshing(.some(current))` | `[.refresh]` |
| `showing(_)` | `descriptorWasReceived(nil)` | `unavailable` | `[]` |
| `showing(_)` | `descriptorWasReceived(.some(descriptor))` | `showing(descriptor)` | `[]` |
| `refreshing(current)` | `onAppear` | `refreshing(current)` | `[]` |
| `refreshing(current)` | `refreshWasRequested` | `refreshing(current)` | `[]` |
| `refreshing(_)` | `descriptorWasReceived(nil)` | `unavailable` | `[]` |
| `refreshing(_)` | `descriptorWasReceived(.some(descriptor))` | `showing(descriptor)` | `[]` |
| `unavailable` | `onAppear` | `unavailable` | `[]` |
| `unavailable` | `refreshWasRequested` | `refreshing(nil)` | `[.refresh]` |
| `unavailable` | `descriptorWasReceived(nil)` | `unavailable` | `[]` |
| `unavailable` | `descriptorWasReceived(.some(descriptor))` | `showing(descriptor)` | `[]` |

## Why this is a Mealy machine

- transition decisions depend on both current state and incoming event
- effects are emitted as data, not performed inside `reduce`
- observation and refresh re-enter the machine as later events
- the enum states make the full workflow table inspectable

## Runtime contract

The runtime story stays small and explicit:

| Effect | Runtime owner | Interpreter shape |
|---|---|---|
| `.startObserving` | store task slot | `AsyncStream<EnergyConsumptionEvent>` |
| `.refresh` | store task slot | `EnergyConsumptionEvent?` |

Using the canonical feature runtime:

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

The store owns task slots and feeds executor output back into `send(_:)`. The canonical reference shows the full runtime wiring.

## Smells

- side effects performed inside `reduce`
- cancellation policy hidden in helper closures instead of the workflow model
- multiple booleans encoding mutually exclusive workflow states
- a separate “fast path” API instead of another branch in the machine
