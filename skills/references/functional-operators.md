# Functional Operators

Use operators to keep pure transformations concise, not to hide intent.

## Defaults

- `map` transforms a value while preserving the container.
- `flatMap` chains operations that already return a container.
- `compactMap` transforms and drops missing values.
- `reduce(into:)` is the default fold when building collections.
- Keep transformations referentially transparent in the pure core.

## Examples

```swift
import Foundation

// Given:
// let people: [Person]
// let rawInput: String?
// let events: [Event]
// let initialState: State
// func reduce(_ state: State, _ event: Event) -> Transition<State, Effect>

let names = people.map(\.name)

let validated = rawInput
    .map { $0.trimmingCharacters(in: .whitespacesAndNewlines) }
    .flatMap { $0.isEmpty ? nil : $0 }

enum ParseFailure: Error {
    case invalid
}

let parsed = Result<Int, ParseFailure>.success(21)
    .map { $0 * 2 }

let byID = people.reduce(into: [UUID: Person]()) { partialResult, person in
    partialResult[person.id] = person
}

let finalState = events.reduce(initialState) { state, event in
    reduce(state, event).state
}
```

## Readability rules

- Keep pipelines short.
- Extract named helpers when closures stop being obvious.
- Do not hide side effects inside `map` or `flatMap`.
- Prefer `reduce(into:)` over manual loops when building dictionaries or sets.
- Use `Result` and `Optional` transformations to keep validation pipelines explicit.

## Good use cases

- value normalization
- projection from domain to derived state
- replaying events through a pure reducer
- fail-fast validation pipelines over `Optional` or `Result`

## Bad use cases

- starting tasks or IO inside transformations
- long chains that mix validation, branching, and side effects
