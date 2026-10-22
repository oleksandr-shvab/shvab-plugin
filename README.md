# shvab

Personal Claude Code plugin: the skills and agents I use as a developer in every project. It is installed once at user scope, so everything here is available in any repo.

## Overview

| Component | Location | Purpose |
|---|---|---|
| Plugin manifest | `.claude-plugin/plugin.json` | Name, version, author |
| Marketplace manifest | `.claude-plugin/marketplace.json` | Lets this repo act as its own marketplace, so it can be installed globally |
| Skills | `skills/<name>/SKILL.md` | Reusable instructions Claude loads when a task matches |
| Agents | `agents/<name>.md` | Subagents for focused, delegated work |

## Skills

| Skill | Purpose |
|---|---|
| `karpathy-guidelines` | Behavioral guidelines to reduce common LLM coding mistakes: think before coding, simplicity first, surgical changes, goal-driven execution |
| `engineering-insights` | Append a non-obvious lesson to a module's `LEARNINGS.md` at the end of a session |
| `implement-plan` | Run implementer → plan-verifier → architecture-reviewer on an existing `plans/NN-slug.md` |
| `react-best-practices` | Modern React conventions and anti-pattern catalog |
| `react-testing-library` | React Testing Library + Vitest guide: setup, queries, async, mocking |
| `zod` | Zod schema validation best practices |

Agents: none yet. Components are added one at a time.

## Install (user scope, all projects)

```
claude plugin marketplace add /home/alexandr/Developer/shvab-plugin
claude plugin install shvab@shvab-marketplace
```

## Develop

Try changes without installing:

```
claude --plugin-dir /home/alexandr/Developer/shvab-plugin
```

Validate manifests and components:

```
claude plugin validate .
```

After changing an installed plugin, bump `version` in `plugin.json` and run:

```
claude plugin update shvab@shvab-marketplace
```

## Adding components

- **Skill:** create `skills/<name>/SKILL.md` with `name` and `description` frontmatter, followed by the instructions.
- **Agent:** create `agents/<name>.md` with `name`, `description` and optional `tools` frontmatter, followed by the system prompt.

## Layout

```
.claude-plugin/
  plugin.json
  marketplace.json
skills/
agents/
.gitignore
README.md
```

## Credits

- `engineering-insights`, `implement-plan`, `react-best-practices`, `react-testing-library` and `zod` are copied unchanged from the local `dev-digest` project (`.claude/skills/`). `zod` originates from [pproenca/dot-skills](https://github.com/pproenca/dot-skills) (per dev-digest's `skills-lock.json`).
- `skills/karpathy-guidelines/` is copied unchanged from [forrestchang/andrej-karpathy-skills](https://github.com/forrestchang/andrej-karpathy-skills) (MIT, per the skill's frontmatter), which is based on [Andrej Karpathy's observations](https://x.com/karpathy/status/2015883857489522876) on LLM coding pitfalls.
