# SOLID in Functional Swift

SOLID still applies, but the implementation tools change.

## Single Responsibility

Prefer:

- Small pure functions
- Narrow types with one reason to change
- Separate transition logic from effect execution

Avoid:

- God stores or god state managers
- Catch-all source-of-truth boundaries
- Enums or structs that mix domain data, IO, and rendering concerns

## Open/Closed

Prefer:

- Extending behavior through composition
- New reducers, policies, or capabilities
- Factories that assemble alternatives without changing the core contract

Avoid:

- Subclass trees for business variation
- Large `switch` statements spread across unrelated layers

## Liskov Substitution

In functional architecture, substitution is mostly about honoring capability contracts.

Prefer:

- Dependency closures with precise semantics
- Capability structs whose functions obey the same invariants in live and test wiring

Avoid:

- Test doubles that succeed where live code can fail, or vice versa
- Protocol abstractions whose implementations disagree on ownership or isolation semantics

## Interface Segregation

Prefer:

- Tiny capability structs
- Small closure-based dependencies
- Feature-specific protocols at external boundaries only

Avoid:

- Fat service protocols
- Shared "environment" objects that expose dozens of unrelated operations

## Dependency Inversion

Prefer:

- Domain and orchestration code depending on capabilities
- Concrete implementations assembled at the composition root

Avoid:

- Domain logic that reaches directly into networking or persistence APIs
- Reducers that know concrete source-of-truth internals

## Example: composition beats inheritance

```swift
import Foundation

struct Money: Sendable, Equatable {
    let amount: Decimal
}

enum PricingPolicy {
    static func applyDiscount(
        _ price: Money,
        percentage: Int
    ) -> Money {
        let factor = Decimal(100 - percentage) / 100
        return Money(amount: price.amount * factor)
    }
}
```

The behavior is extended by adding new policies or composing functions, not by subclassing `Money`.

## Practical heuristics

- Prefer enums as namespaces for stateless business helpers.
- If a dependency list is large, regroup by capability, not by layer name.
- If a type needs inheritance to express workflow variation, first test whether an enum or state machine models it more clearly.
