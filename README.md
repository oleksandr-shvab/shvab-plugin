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
| `engineering-insights` | Append a non-obvious lesson to the touched module's `LEARNINGS.md` (created if the project has none) at the end of a session |
| `implement-plan` | Run implementer → plan-verifier → an optional reviewer fix loop on an existing plan file (`/shvab:implement-plan <plan> [--auto] [--reviewer <agent>]`) |
| `react-best-practices` | React conventions and anti-pattern catalog: state, hooks, component API design, forms, accessibility. Reference files for SSR/Next.js and Motion |
| `react-testing-library` | React Testing Library + Vitest guide: setup, queries, async, mocking |
| `zod` | Zod schema validation best practices |
| `be-review` | Manual Django PR review of the current branch (`/shvab:be-review`): dispatches `flows-review`, then checks house rules — thin views/thick models, QuerySet methods, no direct `save()`, N+1, tests per endpoint |
| `bp` | Manual deep best-practices research on a programming topic (`/shvab:bp <topic>`) |
| `i-have-adhd` | Manual output-shaping mode (`/shvab:i-have-adhd`, off with "stop adhd mode"): lead with the next action, numbered steps, one next action, no preamble/recap/closers |
| `no-ai-slop` | Edit a draft into sharper, more human writing while keeping your voice, or detect AI-slop patterns without rewriting (`/shvab:no-ai-slop <draft>`) |

## Agents

| Agent | Purpose |
|---|---|
| `spec-creator` | Turns a request into a spec (EARS acceptance criteria, `[NEEDS CLARIFICATION]` instead of guessing) |
| `implementation-planner` | Writes a development plan from a spec or a clear request |
| `implementer` | Executes a plan, loading the skills the plan assigns per file |
| `plan-verifier` | Gate that checks the implementation against the plan |
| `test-writer` | Writes tests for existing code, following the project's own conventions |
| `researcher` | Fact-finding with evidence, in the repo and/or on the web; never edits files |
| `flows-review` | Read-only flow-level PR review: traces call hierarchies and finds logical inconsistencies between connected user flows (Django-flavoured); used by `be-review` |

### Pipeline

```
idea ─► spec-creator ─► spec ─┐
                              ├─► implementation-planner ─► plan ─► implementer ─► code
small, clear request ─────────┘                                         │
                                                                        ▼
                                                              plan-verifier (gate)
                                                      INCOMPLETE ◄──────┴──────► PASS
                                                           │                       │
                                                   back to implementer      review · test-writer
```

- **Handoffs are files.** Agents start with no memory of the conversation, so the spec and plan files carry intent from one stage to the next. `/shvab:implement-plan` runs implementer → plan-verifier → review in one command, starting from an existing plan.
- **Project conventions win.** The agents read the project's `CLAUDE.md`/`AGENTS.md`, module docs and `LEARNINGS.md` (if you keep them — see `engineering-insights`), and write specs and plans to the project's own folders. With no convention they default to `specs/NN-slug.md` and `plans/NN-slug.md`.
- **No stack assumptions.** There is no hard-coded module map, command set or skill routing table. Agents find skills at runtime (the skill listing, `.claude/skills/`, `~/.claude/skills/`, installed plugins) and test/typecheck commands from the plan, or else from the module's own scripts.
- **Names are namespaced** once installed: agents as `shvab:<agent>`, skills as `/shvab:<skill>`.
- **Models:** `spec-creator` and `implementation-planner` run on `opus`, the rest on `sonnet` — change it in each agent's frontmatter.

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

- The agents above and the skills `engineering-insights` and `implement-plan` were adapted from the local `dev-digest` project (`.claude/agents/`, `.claude/skills/`) and generalized: dev-digest's paths, commands, MCP tools, sibling skills and skill-routing table were replaced with discovery of each project's own conventions. The steps, gates and "must not do" rules are kept.
- `react-testing-library` and `zod` come from `dev-digest` too. `react-testing-library` got a short "Adapt to the Project First" section plus TypeScript/Jest/router notes; `zod` is unchanged and originates from [pproenca/dot-skills](https://github.com/pproenca/dot-skills) (per dev-digest's `skills-lock.json`).
- `react-best-practices` started as a copy from `dev-digest`, had its project-specific lines (data layer, styling, HTTP client, bundling) generalized, and was extended with rules distilled from personal notes on Josh Comeau's *The Joy of React* course (component API design, state immutability and ownership, forms, hooks gotchas, SSR/Next.js, Motion). The doubtful or version-sensitive claims (Next.js `ssr: false` and segment config, Motion layout and reduced-motion APIs, React 19 context, `defaultProps` and `autoFocus`) were checked against the current docs; the rest comes from the notes.
- `be-review`, `bp` and the `flows-review` agent are copied from the `osh` plugin (v0.1.3, by Oleksandr Shuryhin; marketplace `digital-brother/osh-claude-plugins`). The two commands became skills (same `/shvab:` invocation) and the text is otherwise verbatim — the only edit is `be-review`'s agent dispatch, which names `shvab:flows-review`. The installed copy has no license file and the source repo isn't visible to this machine's `gh` login (likely private), so keep this repo private, or ask the author, before publishing it.
- `skills/i-have-adhd/` is copied unchanged from [ayghri/i-have-adhd](https://github.com/ayghri/i-have-adhd) (MIT, commit `1ecea19`; license text kept next to the skill). Only the skill is taken: the upstream repo's optional always-on `SessionStart` hook is not included, so nothing runs at session start.
- `skills/no-ai-slop/` (with its `eval.md`) is copied unchanged from [petergyang/no-ai-slop](https://github.com/petergyang/no-ai-slop) (MIT, commit `000650b`; license text kept next to the skill).
- `skills/karpathy-guidelines/` is copied unchanged from [forrestchang/andrej-karpathy-skills](https://github.com/forrestchang/andrej-karpathy-skills) (MIT, per the skill's frontmatter), which is based on [Andrej Karpathy's observations](https://x.com/karpathy/status/2015883857489522876) on LLM coding pitfalls.
