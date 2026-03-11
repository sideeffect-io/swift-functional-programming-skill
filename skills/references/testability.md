# Testability

Architecture is testability made visible.

## Test each layer differently

| Layer | What to test |
|---|---|
| Domain values | invariants, parsing, equality, error cases |
| Pure policies | input/output examples and edge cases |
| Reducers or state machines | `state + event -> new state + effects` |
| Effect executors | `effect + fake dependencies -> event/result` |
| Repositories | merge, reconciliation, and source-of-truth rules |
| Factories | light smoke tests only when the wiring is non-trivial |

## Preferred strategy

- Test pure logic directly with values.
- Use closure fakes instead of mock objects.
- Keep dependency surfaces narrow so tests override only what they need.
- Test repository merge rules separately from feature-state behavior.
- Test effect executors separately from reducers.

## What strict concurrency changes

- `Sendable` becomes part of the public contract of your seams.
- Async effect executors should return values small enough to assert directly.
- Tests should not rely on sleeps for sequencing.
- Prefer continuations, `AsyncStream.makeStream`, or explicit fake capabilities to drive async workflows deterministically.

## Example seams

- Reducer tests should never need live infrastructure.
- `@concurrent` executors should be testable with in-memory closure dependencies.
- Repository tests should use an in-memory backing store or a fake DAO boundary rather than full system integration by default.

## Smells

- A test needing a whole object graph to exercise one business rule
- Assertions on private task timing instead of observable events or state
- Protocol mocks whose only purpose is compensating for an oversized dependency surface
- Integration-only coverage for logic that could be pure
