# Swift Functional Architecture

This repository packages the `swift-functional-architecture` skill for AI coding assistants.
The actual skill payload lives under `skills/`.

The skill is now focused on Swift 6.2+ architecture:

- layers and dependency direction
- domain modeling and immutability
- algebraic data types, composition, effects as data, and validation
- reducers and state machines
- dependency injection and composition roots
- strict concurrency boundaries
- testability

It also includes guidance for thin stores as orchestration shells, per-workflow effect executors, and store factories that bind immutable context.

## Layout

- `skills/SKILL.md`: entrypoint and reference map
- `skills/agents/openai.yaml`: UI metadata
- `skills/references/`: focused reference files loaded on demand

## Reference files

- `layers-and-boundaries.md`
- `new-feature-playbook.md`
- `state-management-source-of-truth-factory-boundaries.md`
- `orchestration-shells-effect-executors-and-factories.md`
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

### Manual

Copy or symlink the repository's [`skills/`](./skills) directory into the skills directory used by your agent, under the name `swift-functional-architecture`.

Useful references:

- Agent Skills format: [agentskills.io](https://agentskills.io/)
- Codex skills docs: [developers.openai.com/codex/skills](https://developers.openai.com/codex/skills/)
- Claude Code skills docs: [platform.claude.com/docs/en/agents-and-tools/agent-skills/overview](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)

Example using a copy:

```bash
mkdir -p ~/.codex/skills
cp -R /path/to/this/repo/skills \
  ~/.codex/skills/swift-functional-architecture
```

Example using a symlink:

```bash
mkdir -p ~/.codex/skills
ln -s /path/to/this/repo/skills \
  ~/.codex/skills/swift-functional-architecture
```

### npx

If you use the open Agent Skills CLI, install directly from GitHub with:

```bash
npx skills add https://github.com/sideeffect-io/swift-functional-programming-skill \
  --skill swift-functional-architecture
```

Useful variants:

```bash
# Install globally for your user
npx skills add https://github.com/sideeffect-io/swift-functional-programming-skill \
  --skill swift-functional-architecture -g

# Install specifically for Codex and Claude Code
npx skills add https://github.com/sideeffect-io/swift-functional-programming-skill \
  --skill swift-functional-architecture \
  -a codex -a claude-code
```

### Codex

Install the skill folder into either:

- Project scope: `.codex/skills/`
- User scope: `~/.codex/skills/`

The installed folder should look like:

```text
.codex/
  skills/
    swift-functional-architecture/
      SKILL.md
      agents/
      references/
```

After installation, invoke it explicitly with `$swift-functional-architecture`, or let Codex select it when the task matches the skill description.

### Claude Code

Install the skill folder into either:

- Project scope: `.claude/skills/`
- User scope: `~/.claude/skills/`

The installed folder should look like:

```text
.claude/
  skills/
    swift-functional-architecture/
      SKILL.md
      agents/
      references/
```

After installation, invoke it explicitly with `/swift-functional-architecture`, or let Claude Code select it automatically when relevant.

## License

Apache 2.0. See `LICENSE`.
