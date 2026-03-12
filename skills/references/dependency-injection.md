# Dependency Injection

Prefer the smallest boundary that preserves clarity.

## Order of preference

1. Closures
2. Capability structs
3. Protocols at external boundaries
4. Actors only when stateful isolation is inherent

## Closures first

Use direct closures when a unit needs one or two operations.

```swift
import Foundation

struct Person: Sendable, Equatable {
    let id: UUID
    let name: String
}

enum SavePersonFailure: Error, Sendable, Equatable {
    case persistenceUnavailable
}

enum SavePersonUseCase {
    static func make(
        save: @escaping @Sendable (Person) async throws -> Void,
        reportFailure: @escaping @Sendable (Person, SavePersonFailure) async -> Void
    ) -> @Sendable (Person) async -> Void {
        { person in
            do {
                try await save(person)
            } catch {
                await reportFailure(person, .persistenceUnavailable)
            }
        }
    }
}
```

## Capability structs scale better than long parameter lists

```swift
import Foundation

struct Credentials: Sendable, Equatable {
    let username: String
    let password: String
}

struct Session: Sendable, Equatable {
    let token: String
}

struct AuthenticationClient: Sendable {
    let authenticate: @Sendable (Credentials) async throws -> Session
    let clear: @Sendable () async -> Void
}
```

Use capability structs when:

- several operations belong to one boundary
- you want named overrides in tests
- call sites become noisy with many standalone closures

## Named effect executors are the default seam for effectful feature workflows

Use named effect executors when a reducer or state machine emits effects as data and a thin store or shell needs to interpret them.

Default rules:

- use one executor per workflow or observation
- prefer a concrete `Sendable` struct over a protocol
- expose `callAsFunction`
- depend on narrow closures or capability structs
- return `AsyncStream<Event>` for observation, `Event?` for one-shot workflows, and `Void` for fire-and-forget commands

```swift
import Foundation

enum AuthenticationEvent: Sendable, Equatable {
    case sessionStarted(Session)
    case sessionFailed(message: String)
}

struct StartSessionEffectExecutor: Sendable {
    let authenticate: @Sendable (Credentials) async throws -> Session

    @concurrent
    func callAsFunction(_ credentials: Credentials) async -> AuthenticationEvent? {
        do {
            let session = try await authenticate(credentials)
            return .sessionStarted(session)
        } catch {
            return .sessionFailed(message: String(describing: error))
        }
    }
}
```

This keeps effect interpretation explicit without forcing a protocol or a large dependency bag onto the store.

## Factories can bind immutable context

Factories are the right place to bind immutable feature parameters such as identifiers, route parameters, or parent-owned callbacks.

```swift
import Foundation

struct DeviceStoreFactory: Sendable {
    let observeDevice: @Sendable (UUID) -> AsyncStream<DeviceEvent>
    let refreshDevice: @Sendable (UUID) async throws -> DeviceSnapshot

    @MainActor
    func make(deviceID: UUID) -> DeviceStore {
        DeviceStore(
            observeDevice: .init(observe: { observeDevice(deviceID) }),
            refreshDevice: .init(refresh: { try await refreshDevice(deviceID) })
        )
    }
}
```

Prefer factories that return ready-to-use feature-state managers over factories that return an intermediate dependency bag when the assembled feature is the clearer seam.

## Protocols are boundary tools

Prefer protocols only when you actually need:

- runtime polymorphism across modules
- framework integration that expects reference types
- a public extension point for other modules

If the core does not need polymorphism, adapt the protocol into closures at the boundary and keep the core protocol-free.

## Composition root assembly

- Factories build concrete closure and capability dependencies.
- Factories bind immutable context and assemble effect executors when the feature needs them.
- The core consumes those capabilities without caring how they were produced.
- Tests override only the functions they need.

## Concurrency rules for DI

- Dependency closures that may cross isolation boundaries should be `@Sendable`.
- Use `@concurrent` only on effect executors whose semantics require intentionally leaving caller isolation.
- Do not wrap every async closure in an actor just to make concurrency feel explicit.

## Smells

- One giant environment object shared by unrelated features
- Protocols created only to make mocking possible
- Dependencies passed through layers untouched for no reason
- Hidden global singletons instead of explicit wiring
