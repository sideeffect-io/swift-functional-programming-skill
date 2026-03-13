# Layers and Boundaries

Treat architecture as a dependency-direction problem first.
The running example is `EnergyConsumption`.

## Target layering model

| Layer | Responsibility | `EnergyConsumption` example |
|---|---|---|
| Inert domain | immutable entities, value objects, identifiers, errors | `DeviceIdentifier`, `Device`, `EnergyConsumptionDescriptor` |
| Pure decision logic | reducers, state machines, projections, policies | `EnergyConsumptionProjection`, `EnergyConsumptionStateMachine` |
| Application orchestration | runtime state, event handling, effect launching | `EnergyConsumptionStore` |
| Composition boundary | concrete wiring and immutable context binding | `EnergyConsumptionStoreFactory` |
| Infrastructure and source of truth | persistence, networking, hardware, observation | `DeviceRepository` |

## Dependency direction

- Domain depends on nothing.
- Pure decision logic depends on domain only.
- Application orchestration depends on domain, pure logic, and capabilities.
- Composition depends on everything because it assembles everything.
- Infrastructure depends on concrete systems and returns domain-focused capabilities upward.

If a dependency arrow points inward from a lower layer to a higher one, the boundary is wrong.

## Communication rules

- domain communicates through values
- pure logic communicates through parameters and return values
- orchestration layers talk to infrastructure through capabilities, not direct SDK calls
- source-of-truth boundaries expose domain-focused reads and writes
- factories construct collaborators, they do not decide business behavior

## Source-of-truth rule

Put long-lived authoritative state in a source-of-truth boundary.
Keep optimistic or pending state in the orchestration layer only as long as needed.
Derived presentation state should be recomputed from authoritative state plus explicit transient intent.

## Smells

- a domain type importing framework APIs
- a reducer starting tasks
- a store reading raw transport or persistence payloads
- a factory deciding feature behavior instead of wiring collaborators
