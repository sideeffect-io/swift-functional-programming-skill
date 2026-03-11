# Optics

Optics are optional. Reach for them only when immutable nested updates become repetitive.

## Default stance

- Use plain reconstruction first.
- Use `KeyPath` freely for read access.
- Introduce a tiny lens only when nested immutable updates are frequent enough to justify the abstraction.

## Minimal lens

```swift
import Foundation

struct Lens<Whole, Part> {
    let get: (Whole) -> Part
    let set: (Part, Whole) -> Whole

    func modify(_ transform: (Part) -> Part) -> (Whole) -> Whole {
        { whole in
            let part = get(whole)
            return set(transform(part), whole)
        }
    }
}
```

## KeyPath-backed getter

```swift
import Foundation

extension Lens {
    static func fromKeyPath(
        _ keyPath: KeyPath<Whole, Part>,
        set: @escaping (Part, Whole) -> Whole
    ) -> Lens<Whole, Part> {
        Lens(
            get: { $0[keyPath: keyPath] },
            set: set
        )
    }
}
```

## Use optics when

- reducer state is deep and updated often
- the same nested transformation appears in several places
- the team already understands the abstraction

## Avoid optics when

- state is shallow
- updates are rare
- the abstraction would be harder to read than reconstructing the value directly
