# Validation and Error Modeling

Functional code becomes easier to reason about when success and failure are explicit in the types.

## Optional for absence

Use `Optional` when the only question is whether a value exists.

```swift
import Foundation

func nonEmpty(_ value: String) -> String? {
    value.isEmpty ? nil : value
}

let username = rawUsername
    .map { $0.trimmingCharacters(in: .whitespacesAndNewlines) }
    .flatMap(nonEmpty)
```

## Result for domain failures

Use `Result` when the reason for failure matters.

```swift
import Foundation

enum ParseFailure: Error, Sendable, Equatable {
    case invalidNumber
    case outOfRange
}

func parsePercentage(_ raw: String) -> Result<Int, ParseFailure> {
    guard let value = Int(raw) else { return .failure(.invalidNumber) }
    guard (0...100).contains(value) else { return .failure(.outOfRange) }
    return .success(value)
}
```

Then keep the pipeline explicit:

```swift
let doubled = parsePercentage("21").map { $0 * 2 }
```

## Prefer domain errors over strings

Finite error enums give you:

- better tests
- exhaustive handling
- clearer boundaries

## Fail fast by default

Most business logic is clearer when the first invalid step stops the pipeline.

```swift
import Foundation

enum RegistrationFailure: Error, Sendable, Equatable {
    case emptyUsername
    case invalidAge
}

func validateUsername(_ raw: String) -> Result<String, RegistrationFailure> {
    let trimmed = raw.trimmingCharacters(in: .whitespacesAndNewlines)
    return trimmed.isEmpty ? .failure(.emptyUsername) : .success(trimmed)
}

func validateAge(_ raw: Int) -> Result<Int, RegistrationFailure> {
    raw >= 18 ? .success(raw) : .failure(.invalidAge)
}
```

## Keep validation near the edge

- Parse and validate raw input before it spreads through the system.
- Convert infrastructure failures into domain failures at explicit boundaries.
- Avoid letting invalid primitive values propagate and hoping later layers clean them up.

## Smells

- validation hidden in view code only
- raw strings used as error transport across layers
- boolean return values where the failure reason matters
