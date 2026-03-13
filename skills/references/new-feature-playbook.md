# New Feature Playbook

Use this playbook when the task is to build a feature as a whole rather than only refine one layer.
The running example is `EnergyConsumption`.

## Shared example

- source of truth: `DeviceRepository.observeByID(identifier)` and `refreshByID(identifier)`
- pure projection: `EnergyConsumptionProjection.descriptor(from:)`
- state machine: `EnergyConsumptionStateMachine.reduce`
- effect executors: `ObserveEnergyConsumptionEffectExecutor`, `RefreshEnergyConsumptionEffectExecutor`
- store: `EnergyConsumptionStore`
- composition boundary: `EnergyConsumptionStoreFactory`
- view: `EnergyConsumptionView`

## Build order

1. Define the feature boundary: what it owns, observes, commands, and excludes.
2. Model the domain: entities, identifiers, workflow states, events, and domain errors.
3. Choose the source of truth: what is authoritative, what is derived, what transient state is allowed.
4. Write the pure core first: reducer or state machine, merge rules, policies.
5. List effects as data and choose executor shapes: `AsyncStream<Event>`, `Event?`, or `Void`.
6. Decide whether the feature needs a thin store for lifecycle hooks, task slots, or ergonomic intents.
7. Add dependency injection and composition: closures first, capability structs second, protocols only at real outer boundaries.
8. Add tests by layer.

## Shape heuristics

Use a store over a reducer or state machine when:

- the consumer needs a simpler intent API
- the feature needs lifecycle hooks or task ownership
- the shell can stay thin and act as runtime over named effect executors

Use a direct state machine or state stream when:

- the workflow is the feature
- the public state stream already matches the consumer needs
- a shell would only forward events and states unchanged

## Final checklist

- Is the source of truth explicit?
- Is the feature-state shape justified?
- If this feature uses a store, is it only an orchestration shell?
- Are reducers and state machines pure?
- Are side effects represented explicitly?
- Are dependency seams narrow and `Sendable` where appropriate?
- Are factories wiring only, not deciding business behavior?
- Are tests aligned with the layer responsibilities?
