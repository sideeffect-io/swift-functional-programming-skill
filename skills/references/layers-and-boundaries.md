# Layers and Boundaries

This skill treats architecture as a dependency-direction problem first.

## The target layering model

### 1. Inert domain

- Immutable entities, value objects, identifiers, and errors
- No storage, networking, clocks, tasks, or framework callbacks
- Encodes invariants in types and initializers

### 2. Pure decision logic

- Reducers, merge rules, classifiers, normalization, and policies
- Deterministic input to output functions
- Emits effects as data instead of performing them

### 3. Application orchestration and feature-state management

- Stores, directly consumed state machines, coordinators, and workflow shells
- Receives events, applies pure transitions, launches effects, feeds results back
- Owns transient application state such as pending commands or in-flight tasks

### 4. Composition boundary

- Factories, bootstrap code, and environment assembly
- Builds concrete dependencies and connects features together
- Owns top-level wiring and lifecycle start-up

### 5. Infrastructure

- Repositories, clients, persistence, process boundaries, and platform adapters
- Talks to databases, networks, files, hardware, and operating system services
- Converts low-level IO into capabilities that upper layers can consume

## Dependency direction

- Domain depends on nothing.
- Pure decision logic depends on domain only.
- Application orchestration depends on domain, pure logic, and abstract capabilities.
- Composition depends on everything because it assembles everything.
- Infrastructure depends outward on concrete systems and inward on domain contracts.

If a dependency arrow points the other way, the boundary is wrong.

## Communication rules

- Domain communicates with the rest of the system through values.
- Pure logic communicates through function parameters and return values.
- Feature-state managers communicate with infrastructure through capabilities, not direct SDK calls.
- Repositories communicate upward through domain values and observation streams.
- Prefer the narrowest observation contract at the boundary. Examples may use `AsyncStream` as a standard-library concrete type, but boundary docs should make buffering, termination, and consumer semantics explicit.
- Factories communicate by constructing concrete dependencies, not by carrying business rules.

## Source of truth

- Put long-lived authoritative state in repositories or another explicit domain boundary.
- Keep optimistic or pending application state in orchestration layers only as long as needed.
- Derived presentation or orchestration state should be recomputed from authoritative state plus explicit transient intent, never become a second hidden source of truth.

## Example: composition root wiring

```swift
import Foundation

struct Device: Sendable, Equatable {
    let id: UUID
    let name: String
}

struct DeviceRepository: Sendable {
    let observe: @Sendable () -> AsyncStream<[Device]>
    let refresh: @Sendable () async throws -> Void
}

@MainActor
final class DashboardStore {
    struct Dependencies: Sendable {
        let observeDevices: @Sendable () -> AsyncStream<[Device]>
        let refreshDevices: @Sendable () async throws -> Void
    }

    init(dependencies: Dependencies) {
        self.dependencies = dependencies
    }

    private let dependencies: Dependencies
}

enum DashboardStoreFactory {
    @MainActor
    static func make(repository: DeviceRepository) -> DashboardStore {
        DashboardStore(
            dependencies: .init(
                observeDevices: repository.observe,
                refreshDevices: repository.refresh
            )
        )
    }
}
```

What matters in the example:

- The store does not know how the repository is implemented.
- The repository is the boundary that owns observation and refresh capabilities.
- The factory is the only place where the store sees the concrete repository.
- The same boundary could also expose an async state stream directly instead of wrapping it in a store. The role matters more than the type name.
- `AsyncStream` is only a concrete example here. Do not let public boundaries inherit unbounded buffering or single-consumer assumptions accidentally.

## Smells

- A domain type importing framework APIs
- A reducer starting tasks
- A feature-state manager reading raw SQL rows or HTTP payloads
- A factory deciding feature behavior instead of only wiring collaborators
- An infrastructure type exposed directly to the domain or reducer layer
