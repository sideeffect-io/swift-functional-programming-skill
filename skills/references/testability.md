# Testability

Architecture is testability made visible.
The running example is `EnergyConsumption`.

## Test each layer differently

| Layer | What to test |
|---|---|
| Domain values | invariants, parsing, equality, error cases |
| Pure policies | input/output examples and edge cases |
| Reducers or state machines | `state + event -> new state + effects` |
| Effect executors | `effect + fake dependencies -> event/result` |
| Source-of-truth boundaries | merge, reconciliation, and source-of-truth rules |
| Factories | light smoke tests only when the wiring is non-trivial |

## Preferred strategy

- Test pure logic directly with values.
- Use closure fakes instead of mock objects.
- Keep dependency surfaces narrow so tests override only what they need.
- Test source-of-truth boundary merge rules separately from feature-state behavior.
- Test effect executors separately from reducers.

## What strict concurrency changes

- `Sendable` becomes part of the public contract of your seams.
- Async effect executors should return values small enough to assert directly.
- Tests should not rely on sleeps for sequencing.
- Prefer continuations, `AsyncStream.makeStream`, or explicit fake capabilities to drive async workflows deterministically.

## Example seams

- Reducer tests should never need live infrastructure.
- `ObserveEnergyConsumptionEffectExecutor` should be testable with an in-memory device stream.
- `RefreshEnergyConsumptionEffectExecutor` should be testable with a one-shot fake repository closure.
- `EnergyConsumptionStoreFactory` should only need a light smoke test that proves identifier binding.
- Source-of-truth boundary tests should use an in-memory backing store or a fake lower-level data source rather than full system integration by default.

## Running example: `EnergyConsumption`

- source-of-truth rules, projection, executors, store runtime, and factory wiring are documented in `canonical-feature-energy-consumption.md`
- the exhaustive reducer and transition matrix are documented in `state-machines.md`
- projection tests validate `EnergyConsumptionProjection.descriptor(from:)`
- state-machine tests validate the transition matrix, especially:
  `.idle + .onAppear -> .awaitingFirstSample + [.startObserving]`,
  `.showing(current) + .refreshWasRequested -> .refreshing(.some(current)) + [.refresh]`,
  `.refreshing(_) + .descriptorWasReceived(.some(...)) -> .showing(...)`,
  and `.refreshing(current) + .refreshWasRequested -> .refreshing(current) + []`
- executor tests validate `Device?` observation becomes `.descriptorWasReceived(...)` and one-shot refresh becomes `EnergyConsumptionEvent?`
- store tests validate `start()` idempotence, tracked observation lifetime, and refresh de-duplication
- factory tests validate `make(identifier:)` binds the identifier into the executors

## Smells

- A test needing a whole object graph to exercise one business rule
- Assertions on private task timing instead of observable events or state
- Protocol mocks whose only purpose is compensating for an oversized dependency surface
- Integration-only coverage for logic that could be pure
