# Algebraic Data Types and Totality

Swift does not use FP jargon in the language surface, but `struct` and `enum` are enough to model most domain constraints cleanly.

## Product types

Product types combine values.

```swift
import Foundation

struct Money: Sendable, Equatable {
    let amount: Decimal
    let currencyCode: String
}
```

Use them for domain values that must exist together.

## Sum types

Sum types model alternatives.

```swift
import Foundation

enum LoadState<Value: Sendable & Equatable>: Sendable, Equatable {
    case idle
    case loading
    case loaded(Value)
    case failed(message: String)
}
```

Use them when only one mode may be true at a time.

## Prefer ADTs over boolean piles

If multiple booleans interact, an enum is usually clearer.

Prefer:

```swift
enum CheckoutStep: Sendable, Equatable {
    case cart
    case address
    case payment
    case confirmation(orderID: UUID)
}
```

Over:

- `isEnteringAddress`
- `isPaying`
- `isComplete`

## Total functions

A total function handles every valid input.

```swift
import Foundation

enum ConnectionMode: Sendable, Equatable {
    case local
    case remote
}

enum ConnectionLabel {
    static func text(for mode: ConnectionMode) -> String {
        switch mode {
        case .local: return "Local"
        case .remote: return "Remote"
        }
    }
}
```

Exhaustive pattern matching helps keep the function total as the model evolves.

## Push partiality to the edge

If something can fail, make that explicit in the type.

```swift
import Foundation

struct NonEmptyString: Sendable, Equatable {
    let value: String

    init?(_ value: String) {
        guard value.isEmpty == false else { return nil }
        self.value = value
    }
}
```

This is preferable to accepting invalid values everywhere and patching them later.

## Practical rules

- Use `struct` for stable domain data.
- Use `enum` for alternatives, workflow states, and domain errors.
- Make illegal states unrepresentable when the business rules are stable enough.
- Prefer exhaustive `switch`es over default cases in core business logic.
