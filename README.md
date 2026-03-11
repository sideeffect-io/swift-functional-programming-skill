# Functional Architecture in Swift

This repository packages the `functional-programming-developer` skill for AI coding assistants.
The actual skill payload lives under `skills/`.

The skill is now focused on Swift 6.2+ architecture:

- layers and dependency direction
- domain modeling and immutability
- algebraic data types, composition, effects as data, and validation
- reducers and state machines
- dependency injection and composition roots
- strict concurrency boundaries
- testability

## Layout

- `skills/SKILL.md`: entrypoint and reference map
- `skills/agents/openai.yaml`: UI metadata
- `skills/references/`: focused reference files loaded on demand

## Reference files

- `layers-and-boundaries.md`
- `new-feature-playbook.md`
- `state-management-repository-factory-boundaries.md`
- `solid-in-functional-swift.md`
- `domain-modeling.md`
- `algebraic-data-types-and-totality.md`
- `function-composition.md`
- `effects-as-data.md`
- `validation-and-error-modeling.md`
- `dependency-injection.md`
- `strict-concurrency-boundaries.md`
- `state-machines.md`
- `testability.md`
- `functional-operators.md`
- `optics.md`

## Installation

Codex can load skills from either a per-user directory or a repo-local directory.

Per-user:

```sh
mkdir -p ~/.codex/skills
cp -R /path/to/this/repo/skills ~/.codex/skills/functional-programming-developer
```

Per-repo:

```sh
mkdir -p .codex/skills
cp -R /path/to/this/repo/skills .codex/skills/functional-programming-developer
```

Invoke it explicitly with `$functional-programming-developer`, or let the agent select it when the task matches the skill description.

## License

Apache 2.0. See `LICENSE`.
