# Function Composition

Functional architecture relies on small functions that can be assembled without introducing more objects than necessary.

## Start with plain pure helpers

```swift
import Foundation

enum UsernameRules {
    static func trim(_ value: String) -> String {
        value.trimmingCharacters(in: .whitespacesAndNewlines)
    }

    static func lowercase(_ value: String) -> String {
        value.lowercased()
    }
}
```

Compose these helpers instead of rewriting the full transformation in every call site.

## Composition helper

```swift
import Foundation

func compose<A, B, C>(
    _ f: @escaping (A) -> B,
    _ g: @escaping (B) -> C
) -> (A) -> C {
    { value in g(f(value)) }
}
```

Usage:

```swift
let normalizeUsername = compose(
    UsernameRules.trim,
    UsernameRules.lowercase
)
```

## Partial application

Fix dependencies first, input last.

```swift
import Foundation

func makeGreeting(
    prefix: String
) -> (String) -> String {
    { name in "\(prefix), \(name)" }
}

let greetFormally = makeGreeting(prefix: "Hello")
```

This is useful for dependency injection and configuration-heavy pure logic.

## Higher-order functions

Functions that take or return functions are often enough.

```swift
import Foundation

func mapName(
    _ transform: @escaping (String) -> String
) -> (User) -> User {
    { user in
        User(id: user.id, name: transform(user.name))
    }
}

struct User: Sendable, Equatable {
    let id: UUID
    let name: String
}
```

## Heuristics

- Extract a helper when the transformation has a stable name.
- Compose functions when the pipeline stays obvious.
- Stop and introduce a dedicated type when behavior needs identity, mutable state, or a broader protocol boundary.
- Prefer readable pipelines over clever operator-heavy code.
