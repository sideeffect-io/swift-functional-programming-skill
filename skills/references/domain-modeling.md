# Domain Modeling

Model the problem first. Architecture quality is downstream from type quality.

## Use value types by default

- `struct` for product types
- `enum` for alternatives, workflows, and constrained modes
- Explicit value objects for values with invariants

## Encode invariants in types

```swift
import Foundation

struct Percentage: Sendable, Equatable {
    let value: Int

    init?(_ value: Int) {
        guard (0...100).contains(value) else { return nil }
        self.value = value
    }
}
```

Prefer rejecting invalid values at the edge over letting invalid states leak inward.

## Prefer workflow states over boolean piles

```swift
import Foundation

struct Session: Sendable, Equatable {
    let token: String
}

enum LoginState: Sendable, Equatable {
    case idle
    case submitting(username: String)
    case authenticated(Session)
    case failed(message: String)
}
```

This is clearer than a struct with `isLoading`, `isAuthenticated`, and `errorMessage` that can drift into impossible combinations.

## Separate authoritative state from derived state

```swift
import Foundation

struct DeviceRecord: Sendable, Equatable {
    let isOn: Bool
    let level: Int
}

struct DevicePresentation: Sendable, Equatable {
    let isOn: Bool
    let normalizedLevel: Double
}

enum DeviceProjection {
    static func make(from record: DeviceRecord) -> DevicePresentation {
        DevicePresentation(
            isOn: record.isOn,
            normalizedLevel: Double(record.level) / 100
        )
    }
}
```

The record is authoritative. The projection is derived. Do not persist or hand-edit the derived value as if it were a second source of truth.

## Domain errors should be finite and explicit

Prefer:

```swift
import Foundation

enum ConnectionFailure: Error, Sendable, Equatable {
    case invalidCredentials
    case endpointUnavailable
    case unexpectedResponse
}
```

Over:

- Raw strings
- Opaque error codes flowing through every layer unchanged

## Concrete-state modeling is also valid

For larger workflows, you may prefer one type per state plus a projected aggregate:

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
    let startedAt: Date

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

Choose this style when each state carries distinct data and you want a projected aggregate without a giant switch at every call site.

## Naming rules

- Use nouns for domain values.
- Use past-tense events for observed facts.
- Use imperative names for commands only at effect boundaries.
- Keep workflow state names explicit enough that illegal combinations are obvious.

## Smells

- Multiple optionals whose presence depends on a hidden mode
- A "kind" field plus many unrelated stored properties
- Domain values that know transport, storage, or framework details
- Source-of-truth boundaries or feature-state managers patching invalid domain values after the fact
