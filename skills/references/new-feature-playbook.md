# New Feature Playbook

Use this playbook when the task is to build a feature as a whole rather than only refine one layer.

## 1. Define the feature boundary

State in one paragraph:

- what the feature owns
- what it observes
- what it can command
- what must remain outside the feature

If the boundary is unclear, the implementation will drift.

## 2. Start from the domain

Model first:

- entities and value objects
- identifiers
- workflow states
- events as observed facts
- domain errors

Prefer `struct` and `enum`. Eliminate illegal states before thinking about views or task wiring.

## 3. Choose the source of truth

Decide explicitly:

- what state is authoritative
- what state is derived
- what transient optimistic or pending state is allowed

Authoritative state usually belongs in a source-of-truth boundary or another explicit domain boundary. Transient local state belongs in the feature-state layer.

## 4. Choose the feature-state shape

The feature-state layer is a role, not a mandatory type.

### Use a direct state machine or state stream when

- the workflow is the feature
- the public state stream already matches the consumer needs
- adding a shell would mostly forward events and states unchanged

### Use a store over a reducer or state machine when

- the consumer needs a simpler intent API
- the feature needs lifecycle hooks or task ownership
- you need local pending state, derived state, or effect bookkeeping around the workflow

### Use a thin coordinator shell when

- the feature mostly routes between child flows
- the shell adds orchestration value without becoming a god object

## 5. Define transitions before effects

Write the pure core first:

- reducer or state machine transition
- merge and normalization rules
- policies and classifiers

Only after that, define the effect values the core can emit.

Prefer:

```swift
(State, Event) -> Transition<State, Effect>
```

Over reducers or handlers that call infrastructure directly.

## 6. Define effects as data

List the effect cases explicitly.

For each effect, decide:

- which capability it needs
- whether it should inherit caller isolation or run with `@concurrent`
- which event or result it feeds back into the feature

Keep lifecycle and cancellation policy visible in the workflow model, not hidden inside helpers.

## 7. Define source-of-truth boundaries

Ask:

- does the feature need authoritative persistence?
- does it merge multiple sources?
- does it own cross-aggregate coherence?

If yes, put that logic in a source-of-truth boundary, not in factories or in ad hoc task closures.

## 8. Define dependency injection and composition

At the feature boundary:

- closures first
- capability structs second
- protocols only for real outer-boundary needs

At the app boundary:

- factories assemble concrete dependencies
- factories connect features together
- factories do not hide business rules that belong in the reducer, state machine, or source-of-truth boundary

## 9. Pick the delivery path

### Simple feature path

Use this path when the feature is mostly projection and a few commands.

1. Model domain values
2. Define authoritative source-of-truth inputs and outputs
3. Add a minimal reducer or thin state manager
4. Add effect executors if commands are needed
5. Add targeted tests for pure logic and command mapping

### Workflow-heavy feature path

Use this path when retries, cancellation, parallel flows, or long-lived async behavior matter.

1. Model states, events, and effects explicitly
2. Write the pure state machine first
3. Add interpreters for emitted effects
4. Add an optional shell only if consumer ergonomics or lifecycle require it
5. Test transitions, effect interpreters, and composition seams separately

## 10. Test by layer

- Domain: invariants and parsing
- Pure logic: transitions, merge rules, projections
- Effect executors: `effect -> event/result` with fake capabilities
- Source-of-truth boundaries: authoritative-state rules and reconciliation
- Composition: light smoke tests only when wiring is non-trivial

Do not rely on full integration tests to validate logic that could be pure.

## 11. Final checklist

- Is the source of truth explicit?
- Is the feature-state shape justified?
- Are reducers and state machines pure?
- Are side effects represented explicitly?
- Are dependency seams narrow and `Sendable` where appropriate?
- Are factories wiring only, not deciding business behavior?
- Are source-of-truth boundaries owning persistence and cross-source coherence where needed?
- Are tests aligned with the layer responsibilities?

If any answer is unclear, the feature is not fully designed yet.
