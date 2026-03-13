# Effects as Data

The core move is to separate deciding from doing.
The running example is `EnergyConsumption`.

## Minimal shape

```swift
struct Transition<State: Sendable, Effect: Sendable>: Sendable {
    let state: State
    let effects: [Effect]
}
```

Reducers and state machines return `Transition`, not live IO.

## Canonical example

Using `EnergyConsumptionState`, `EnergyConsumptionEvent`, and `EnergyConsumptionEffect` from `state-machines.md`:

```swift
switch (state, event) {
case (.idle, .onAppear):
    return .init(state: .awaitingFirstSample, effects: [.startObserving])

case (.showing(let current), .refreshWasRequested):
    return .init(state: .refreshing(current), effects: [.refresh])

case (.refreshing(_), .descriptorWasReceived(.some(let descriptor))):
    return .init(state: .showing(descriptor), effects: [])

default:
    return .init(state: state, effects: [])
    }
}
```

The full reducer lives in `state-machines.md`. The point here is the boundary:

- transition logic emits `.startObserving` and `.refresh`
- interpreters later turn those effects into `descriptorWasReceived(...)`
- the machine remains pure even when the feature mixes a stream workflow and a one-shot workflow

## Interpreter shapes

Using the canonical runtime from `canonical-feature-energy-consumption.md`:

| Effect kind | Executor | Return shape |
|---|---|---|
| Observation | `ObserveEnergyConsumptionEffectExecutor` | `AsyncStream<EnergyConsumptionEvent>` |
| One-shot workflow | `RefreshEnergyConsumptionEffectExecutor` | `EnergyConsumptionEvent?` |

## Why this matters

- reducers stay deterministic
- tests assert on emitted effects rather than mocking live IO
- runtime policy stays in the store, not in the reducer
- effect shape stays stable even as the runtime changes

## Smells

- reducers calling repositories directly
- hidden side effects in computed properties or initializers
- workflow rules split between transition logic and interpreter code
