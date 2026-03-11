# Effects as Data

The central FP architecture move is to separate deciding from doing.

## The idea

- Pure logic decides what should happen next.
- That decision is represented as values.
- A boundary interpreter performs the side effects later.

This keeps business rules deterministic and testable.

## Minimal shape

```swift
import Foundation

struct Transition<State: Sendable, Effect: Sendable>: Sendable {
    let state: State
    let effects: [Effect]
}
```

The reducer or policy returns `Transition`, not live IO.

## Example

```swift
import Foundation

enum SaveState: Sendable, Equatable {
    case idle
    case saving
    case saved
    case failed(message: String)
}

enum SaveEvent: Sendable, Equatable {
    case saveWasRequested(Document)
    case saveDidSucceed
    case saveDidFail(message: String)
}

enum SaveEffect: Sendable, Equatable {
    case persist(Document)
}

struct Document: Sendable, Equatable {
    let id: UUID
}

enum SaveMachine {
    static func reduce(
        state: SaveState,
        event: SaveEvent
    ) -> Transition<SaveState, SaveEffect> {
        switch (state, event) {
        case (_, .saveWasRequested(let document)):
            return .init(state: .saving, effects: [.persist(document)])

        case (.saving, .saveDidSucceed):
            return .init(state: .saved, effects: [])

        case (.saving, .saveDidFail(let message)):
            return .init(state: .failed(message: message), effects: [])

        default:
            return .init(state: state, effects: [])
        }
    }
}
```

## Interpreters live at the boundary

```swift
import Foundation

enum SaveInterpreter {
    @concurrent
    static func run(
        _ effect: SaveEffect,
        persist: @escaping @Sendable (Document) async throws -> Void
    ) async -> SaveEvent {
        switch effect {
        case .persist(let document):
            do {
                try await persist(document)
                return .saveDidSucceed
            } catch {
                return .saveDidFail(message: String(describing: error))
            }
        }
    }
}
```

## Why this matters

- reducers stay pure
- tests assert on effect values instead of mocking side effects
- lifecycle and cancellation policy can be modeled explicitly
- the same core logic can be used with different interpreters

## Smells

- reducers that call repositories directly
- hidden side effects in computed properties or initializers
- workflow rules split between transition logic and effect helpers
