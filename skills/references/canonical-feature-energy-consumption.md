# Canonical Feature Example: Energy Consumption

This is the single vertical slice reused across the core references in this skill.
It is authoritative for the runtime side of the feature: source of truth, projection, executors, store, factory, view, and tests.

`EnergyConsumptionState`, `EnergyConsumptionEvent`, `EnergyConsumptionEffect`, and the exhaustive reducer live in `state-machines.md`.

## Feature boundary

- owns feature-local workflow state for one card
- observes authoritative device state from `DeviceRepository`
- can request a one-shot refresh from the same source-of-truth boundary
- does not own persistence, merge rules, or cross-feature coordination

## Source of truth and projection

```swift
struct DeviceIdentifier: Hashable, Sendable {
    let rawValue: UUID
}

struct Device: Sendable, Equatable {
    let id: DeviceIdentifier
    let kilowattHours: Double?
    let unitSymbol: String
}

struct DeviceRepository: Sendable {
    let observeByID: @Sendable (DeviceIdentifier) async -> AsyncStream<Device?>
    let refreshByID: @Sendable (DeviceIdentifier) async -> Device?
}

struct EnergyConsumptionDescriptor: Sendable, Equatable {
    let value: Double
    let unitSymbol: String
}

enum EnergyConsumptionProjection {
    static func descriptor(from device: Device?) -> EnergyConsumptionDescriptor? {
        guard let device, let kilowattHours = device.kilowattHours else {
            return nil
        }

        return .init(value: kilowattHours, unitSymbol: device.unitSymbol)
    }
}
```

The repository owns authoritative state and observation. The descriptor is derived feature data, not a second source of truth.

## Machine recap

The runtime below uses the machine defined in `state-machines.md`:

| Concern | Names |
|---|---|
| States | `idle`, `awaitingFirstSample`, `showing(EnergyConsumptionDescriptor)`, `refreshing(EnergyConsumptionDescriptor?)`, `unavailable` |
| Events | `onAppear`, `refreshWasRequested`, `descriptorWasReceived(EnergyConsumptionDescriptor?)` |
| Effects | `startObserving`, `refresh` |

That file is the authoritative place for the exhaustive reducer and transition matrix.

## Effect executors

```swift
struct ObserveEnergyConsumptionEffectExecutor: Sendable {
    let observeEnergyConsumption: @Sendable () async -> AsyncStream<Device?>

    @concurrent
    func callAsFunction() async -> AsyncStream<EnergyConsumptionEvent> {
        let stream = await observeEnergyConsumption()

        return AsyncStream { continuation in
            let task = Task {
                for await device in stream {
                    continuation.yield(
                        .descriptorWasReceived(
                            EnergyConsumptionProjection.descriptor(from: device)
                        )
                    )
                }
                continuation.finish()
            }

            continuation.onTermination = { _ in
                task.cancel()
            }
        }
    }
}

struct RefreshEnergyConsumptionEffectExecutor: Sendable {
    let refreshEnergyConsumption: @Sendable () async -> Device?

    @concurrent
    func callAsFunction() async -> EnergyConsumptionEvent? {
        guard !Task.isCancelled else { return nil }

        let device = await refreshEnergyConsumption()

        guard !Task.isCancelled else { return nil }

        return .descriptorWasReceived(
            EnergyConsumptionProjection.descriptor(from: device)
        )
    }
}
```

Each executor does one workflow only:

- observation stream -> `AsyncStream<EnergyConsumptionEvent>`
- one-shot refresh -> `EnergyConsumptionEvent?`

## Store runtime

```swift
@Observable
@MainActor
final class EnergyConsumptionStore {
    private(set) var state: EnergyConsumptionState = .idle

    private let observeEnergyConsumption: ObserveEnergyConsumptionEffectExecutor
    private let refreshEnergyConsumption: RefreshEnergyConsumptionEffectExecutor
    private var observationTask: Task<Void, Never>?
    private var refreshTask: Task<Void, Never>?
    private var hasStarted = false

    init(
        observeEnergyConsumption: ObserveEnergyConsumptionEffectExecutor,
        refreshEnergyConsumption: RefreshEnergyConsumptionEffectExecutor
    ) {
        self.observeEnergyConsumption = observeEnergyConsumption
        self.refreshEnergyConsumption = refreshEnergyConsumption
    }

    func start() {
        guard !hasStarted else { return }
        hasStarted = true
        send(.onAppear)
    }

    func refresh() {
        send(.refreshWasRequested)
    }

    func send(_ event: EnergyConsumptionEvent) {
        let transition = EnergyConsumptionStateMachine.reduce(state, event)
        state = transition.state

        for effect in transition.effects {
            handle(effect)
        }
    }

    private func handle(_ effect: EnergyConsumptionEffect) {
        switch effect {
        case .startObserving:
            guard observationTask == nil else { return }
            observationTask = Task { [observeEnergyConsumption] in
                let events = await observeEnergyConsumption()

                for await event in events {
                    await MainActor.run {
                        self.send(event)
                    }
                }

                await MainActor.run {
                    self.observationTask = nil
                }
            }

        case .refresh:
            guard refreshTask == nil else { return }
            refreshTask = Task { [refreshEnergyConsumption] in
                let event = await refreshEnergyConsumption()

                if let event {
                    await MainActor.run {
                        self.send(event)
                    }
                }

                await MainActor.run {
                    self.refreshTask = nil
                }
            }
        }
    }

    deinit {
        observationTask?.cancel()
        refreshTask?.cancel()
    }
}
```

The store is only the runtime shell. It owns task slots, lifecycle, and event forwarding. Transition logic stays in the pure state machine.

## Factory and view

```swift
struct EnergyConsumptionStoreFactory: Sendable {
    let deviceRepository: DeviceRepository

    @MainActor
    func make(identifier: DeviceIdentifier) -> EnergyConsumptionStore {
        EnergyConsumptionStore(
            observeEnergyConsumption: .init(
                observeEnergyConsumption: {
                    await deviceRepository.observeByID(identifier)
                }
            ),
            refreshEnergyConsumption: .init(
                refreshEnergyConsumption: {
                    await deviceRepository.refreshByID(identifier)
                }
            )
        )
    }
}

struct EnergyConsumptionView: View {
    @Environment(\.energyConsumptionStoreFactory) private var energyConsumptionStoreFactory

    let identifier: DeviceIdentifier

    var body: some View {
        WithStoreView(store: energyConsumptionStoreFactory.make(identifier: identifier)) { store in
            VStack(spacing: 12) {
                Button("Refresh") { store.refresh() }

                switch store.state {
                case .idle, .awaitingFirstSample, .refreshing(nil):
                    ProgressView("Loading energy consumption")

                case .showing(let descriptor), .refreshing(.some(let descriptor)):
                    EnergyCard(descriptor: descriptor)

                case .unavailable:
                    ContentUnavailableView("No Energy Data", systemImage: "bolt.slash")
                }
            }
        }
    }
}
```

The factory binds immutable context and assembles the executors. The view asks for a ready-to-use store and renders workflow state directly.

## Test split

- source-of-truth tests validate authoritative observation and refresh behavior
- projection tests validate `Device -> EnergyConsumptionDescriptor`
- state-machine tests live in `state-machines.md` and prove the full transition matrix
- executor tests validate observation -> `AsyncStream<Event>` and refresh -> `Event?`
- store tests validate `start()` idempotence, refresh de-duplication, task ownership, and event forwarding
- factory tests stay light and only verify identifier binding and wiring
