# State Machines

Use a state machine when the workflow, not the view hierarchy, is the hard part.

The state machine can either sit behind a store-like shell or be the feature-state layer itself if the view can consume its state stream directly.

## When a reducer is enough

Use a plain reducer when:

- there are only a few stable states
- effects are simple and independent
- cancellation and supervision rules are minimal

## When to use an extended Mealy machine

Use a state machine when:

- transitions depend on both mode and context
- effects feed results back into the workflow
- lifecycle, cancellation, retries, or parallel flows must be explicit
- you need to prove where orchestration logic lives

The canonical shape is:

```swift
(State, Event) -> Transition<State, Effect>
```

Where `Transition` is pure data:

```swift
import Foundation

struct Transition<State: Sendable, Effect: Sendable>: Sendable {
    let state: State
    let effects: [Effect]
}
```

## Core example

```swift
import Foundation

struct Credentials: Sendable, Equatable {
    let username: String
    let password: String
}

struct Session: Sendable, Equatable {
    let token: String
}

enum AuthState: Sendable, Equatable {
    case idle
    case submitting(Credentials)
    case authenticated(Session)
    case failed(message: String)
}

enum AuthEvent: Sendable, Equatable {
    case submitWasRequested(Credentials)
    case sessionWasStarted(Session)
    case sessionStartDidFail(message: String)
}

enum AuthEffect: Sendable, Equatable {
    case startSession(Credentials)
}

enum AuthMachine {
    static func reduce(
        state: AuthState,
        event: AuthEvent
    ) -> Transition<AuthState, AuthEffect> {
        switch (state, event) {
        case (_, .submitWasRequested(let credentials)):
            return .init(
                state: .submitting(credentials),
                effects: [.startSession(credentials)]
            )

        case (.submitting, .sessionWasStarted(let session)):
            return .init(
                state: .authenticated(session),
                effects: []
            )

        case (.submitting, .sessionStartDidFail(let message)):
            return .init(
                state: .failed(message: message),
                effects: []
            )

        default:
            return .init(state: state, effects: [])
        }
    }
}
```

The reducer is pure. The only side effects are described in `AuthEffect`.

## Effect execution in Swift 6.2+

Prefer named `@concurrent` effect executors over generic static runners when the effect must intentionally leave caller isolation and the effect itself is stateless.

```swift
import Foundation

struct AuthenticationClient: Sendable {
    let authenticate: @Sendable (Credentials) async throws -> Session
}

struct StartSessionEffectExecutor: Sendable {
    let authenticate: @Sendable (Credentials) async throws -> Session

    @concurrent
    func callAsFunction(_ credentials: Credentials) async -> AuthEvent? {
        do {
            let session = try await authenticate(credentials)
            return .sessionWasStarted(session)
        } catch {
            return .sessionStartDidFail(message: String(describing: error))
        }
    }
}
```

## Orchestration shell

The shell owns tasks, not business rules.
It is optional.

Use a shell when the view benefits from:

- a simple intent API
- lifecycle hooks
- local task ownership and cancellation bookkeeping

Skip the shell when the state machine itself already exposes the right state stream and command surface for the feature.
The shell owns tasks and runtime policy; executors own effect interpretation; reducers own workflow rules.

```swift
import Foundation

@MainActor
final class AuthStore {
    private(set) var state: AuthState = .idle
    private let startSession: StartSessionEffectExecutor
    private var runningTasks: [UUID: Task<Void, Never>] = [:]

    init(startSession: StartSessionEffectExecutor) {
        self.startSession = startSession
    }

    func send(_ event: AuthEvent) {
        let transition = AuthMachine.reduce(state: state, event: event)
        state = transition.state

        for effect in transition.effects {
            handle(effect)
        }
    }

    private func handle(_ effect: AuthEffect) {
        let id = UUID()
        runningTasks[id] = Task { [startSession] in
            let nextEvent: AuthEvent?

            switch effect {
            case .startSession(let credentials):
                nextEvent = await startSession(credentials)
            }

            await MainActor.run {
                self.runningTasks[id] = nil
                if let nextEvent {
                    self.send(nextEvent)
                }
            }
        }
    }
}
```

### Tracked observation vs one-shot executors

Use distinct executors when a workflow's runtime shape differs.

```swift
import Foundation

struct ObserveSessionEffectExecutor: Sendable {
    let observe: @Sendable () -> AsyncStream<AuthEvent>

    @concurrent
    func callAsFunction() async -> AsyncStream<AuthEvent> {
        observe()
    }
}

struct RefreshSessionEffectExecutor: Sendable {
    let refresh: @Sendable () async throws -> Session

    @concurrent
    func callAsFunction() async -> AuthEvent? {
        do {
            return .sessionWasStarted(try await refresh())
        } catch {
            return .sessionStartDidFail(message: String(describing: error))
        }
    }
}
```

## Lifecycle and cancellation rules

Model lifecycle in the machine, not inside effect-building helpers.

Prefer:

- explicit events such as `observationWasStarted`, `retryWasRequested`, `cancellationWasRequested`
- explicit effects such as `.startObservation`, `.cancelObservation(id:)`
- cancellation keys or task identifiers owned by the orchestration shell

Avoid:

- ad hoc task cancellation rules hidden in closures
- effect builders that decide workflow semantics on their own

## Enum states vs concrete states

### Enum state

Use enum state when:

- the workflow is small or medium
- pattern matching on states is straightforward
- one type is enough to express the context

### Concrete states plus projected aggregate

Use concrete state types when:

- each state carries very different data
- you want to project a shared aggregate without a large switch everywhere
- you want to add states without constantly editing one giant enum consumer

Example projection style:

```swift
import Foundation

struct SyncProjection: Sendable, Equatable {
    let isRunning: Bool
    let errorMessage: String?
}

struct SyncIsIdle: Sendable, Equatable {
    var superstate: SyncProjection {
        SyncProjection(isRunning: false, errorMessage: nil)
    }
}

struct SyncIsRunning: Sendable, Equatable {
    let progress: Double

    var superstate: SyncProjection {
        SyncProjection(isRunning: true, errorMessage: nil)
    }
}

struct SyncHasFailed: Sendable, Equatable {
    let message: String

    var superstate: SyncProjection {
        SyncProjection(isRunning: false, errorMessage: message)
    }
}
```

## Parent and child coordination

- Keep child workflows reusable and unaware of the parent where possible.
- Parent workflows should translate child outcomes into parent events explicitly.
- Join rules, supervision, and cross-flow coordination belong in the parent workflow or the composition boundary, not in effect executors.

## Streams

When a workflow needs a local event stream for tests or adapters, prefer the standard library API:

```swift
import Foundation

let (events, continuation) = AsyncStream.makeStream(of: AuthEvent.self)
_ = events
_ = continuation
```

That same style can be used as the public feature-state surface when a separate store object would add no value, but do not treat `AsyncStream` as a magic boundary type. Make buffering, termination, and consumer semantics explicit if the stream escapes a local adapter or test seam.

## Smells

- Side effects performed inside `reduce`
- A worker actor added only to call a stateless async function
- Parent-child wiring hidden in global singletons
- Fast path behavior implemented as a separate API instead of a branch in the machine
- Cancellation rules spread across helpers instead of modeled as events and effects
