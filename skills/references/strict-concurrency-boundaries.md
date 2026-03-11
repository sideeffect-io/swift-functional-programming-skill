# Strict Concurrency Boundaries

This skill assumes Swift 6.2+ architecture, but it stays explicit about execution semantics.

## Baseline assumptions

- Strict concurrency matters to the design, not only to compiler cleanup.
- `Sendable` is part of boundary design.
- When an effect truly must leave caller isolation, `@concurrent` is usually clearer than inventing a stateless worker actor.
- `@concurrent` is not a blanket default; use it only when leaving caller isolation is part of the intended semantics.
- Actors remain valid, but only for true isolated mutable state.

## The key Swift 6.2 rule

SE-0461 changes the model for nonisolated async functions when the `NonisolatedNonsendingByDefault` feature is enabled:

- plain nonisolated async functions run on the caller's actor by default
- `@concurrent` functions explicitly switch off that actor and run on the generic executor

Use that distinction intentionally.

## Default choices

### Pure sync helper

- Use a plain pure function.
- If it lives inside an actor- or global-actor-isolated type, mark it `nonisolated` when you need to prevent accidental actor inference.

### Pure or lightweight async helper that should inherit caller isolation

- Use plain async functions in modules that enable `NonisolatedNonsendingByDefault`.
- In reusable examples or migration-sensitive code, spell `nonisolated(nonsending)` explicitly when clarity matters.

### Async effect that should run concurrently with the caller

- Use `@concurrent` sparingly.
- Return an event or a small result value so the orchestration layer can feed the outcome back into the workflow.

### Shared mutable state

- Use an actor, repository, or another explicit serialization boundary.
- Do not introduce an actor only to wrap stateless async functions.

## Example: `@concurrent` effect executor

```swift
import Foundation

struct Command: Sendable, Equatable {
    let name: String
}

enum CommandEvent: Sendable, Equatable {
    case commandSucceeded
    case commandFailed(message: String)
}

enum CommandExecutor {
    @concurrent
    static func run(
        _ command: Command,
        send: @escaping @Sendable (Command) async throws -> Void
    ) async -> CommandEvent {
        do {
            try await send(command)
            return .commandSucceeded
        } catch {
            return .commandFailed(message: String(describing: error))
        }
    }
}
```

## Example: keep pure helpers off the actor

```swift
import Foundation

@MainActor
final class FeatureStore {
    struct Snapshot: Sendable, Equatable {
        let rawLevel: Int
    }

    struct Descriptor: Sendable, Equatable {
        let normalizedLevel: Double
    }

    nonisolated static func makeDescriptor(from snapshot: Snapshot) -> Descriptor {
        Descriptor(normalizedLevel: Double(snapshot.rawLevel) / 100)
    }
}
```

The helper does not need main-actor isolation, so the design says so explicitly.

## Decision table

| Need | Use |
|---|---|
| Pure logic | Plain function |
| Pure helper inside isolated type | `nonisolated` helper |
| Async helper that should inherit caller isolation | Plain async or `nonisolated(nonsending)` |
| Async work that must intentionally leave caller isolation | `@concurrent` |
| Stateful serialized boundary | Actor |

## Smells

- A worker actor with no meaningful protected state
- `@MainActor` used to silence a sendability problem instead of modeling isolation correctly
- Async helpers whose execution semantics are unclear from the signature
- `@unchecked Sendable` used where better boundaries would remove the need
