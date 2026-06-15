# Skills

A curated collection of reusable AI agent skills — procedural knowledge, workflows, and patterns designed for use with [Hermes Agent](https://hermes-agent.nousresearch.com/).

Each skill lives in its own directory and follows the Hermes SKILL.md format (YAML frontmatter + markdown body), making it portable across agent systems that support the convention.

## Skills

| Skill | Description |
|-------|-------------|
| [loop-design](./loop-design/SKILL.md) | Design high-quality agent loops with structured interviews. Prevents loopmaxxing by defining goals, restrictions, deterministic exit conditions, and guardrails before building. |

## Adding a New Skill

1. Create a new directory with a short, descriptive name (kebab-case).
2. Write a `SKILL.md` with YAML frontmatter (name, description, version, author, license, platforms, metadata).
3. Open a pull request.

The frontmatter format:

```yaml
---
name: your-skill-name
description: "One-line description of what this skill does."
version: 1.0.0
author: <your-name>
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [tag1, tag2]
    related_skills: [other-skill-names]
---
```

## License

MIT — see [LICENSE](./LICENSE).
