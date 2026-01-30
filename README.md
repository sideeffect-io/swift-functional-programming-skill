# Functional Architecture in Swift (Agent Skill)

This repository is an agent skill for AI coding assistants. It teaches Swift
functional programming patterns for domain and core logic: immutability, pure
functions, composition, state machines, reducers, dependency injection via functions,
and effects as data.

## What's inside

- `SKILL.md` with the skill metadata and core instructions.
- `references/` with deeper reading:
  - `state-machines.md`
  - `functional-operators.md`
  - `algebraic-data-types.md`
  - `optics.md`
  - `dependency-injection-currying.md`
  - `dependency-injection-decision-table.md`

## Install

This repository already matches the agent skills folder layout (a skills folder
with a required `SKILL.md` and optional supporting folders).

### NPX

`npx skills add sideeffect-io/swift-functional-programming-skill`

### Codex (CLI or IDE)

Codex can load skills from either a per-user directory or a repo-local
directory. After installing, restart Codex.

Per-user:

```sh
mkdir -p ~/.codex/skills
cp -R /path/to/this/repo ~/.codex/skills/functional-programming-developer
```

Per-repo:

```sh
mkdir -p .codex/skills
cp -R /path/to/this/repo .codex/skills/functional-programming-developer
```

Use it by invoking `$functional-programming-developer` or by describing a task
that matches the skill so Codex can select it automatically.

### Claude Code

Claude Code supports skills and can load them automatically when relevant.
For manual installation, place this folder under your Claude Code skills
directory (commonly `~/.claude/skills`).

```sh
mkdir -p ~/.claude/skills
cp -R /path/to/this/repo ~/.claude/skills/functional-programming-developer
```

## Usage

Codex example:

```
$functional-programming-developer
Model this feature with reducers and explicit effects.
```

Claude Code example:

```
Use the Functional Architecture in Swift skill to refactor this domain layer
into pure functions with injected effects.
```

## Contributing

Contributions are welcome. Suggested ways to help:

- Refine the rules in `SKILL.md` while keeping the focus on functional core
  patterns and testability.
- Add or improve reference docs in `references/` and update the reading order
  in `SKILL.md`.
- Keep changes aligned with the agent skills folder structure (`SKILL.md`
  required, optional `references/`, `scripts/`, `assets/`).

Open a PR with a short rationale and examples if applicable.

## License

Apache 2.0. See `LICENSE`.
