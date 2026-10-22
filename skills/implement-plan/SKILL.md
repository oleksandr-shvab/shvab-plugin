---
name: implement-plan
description: "Runs implementer → plan-verifier (gate) → an optional review pass with an auto-fix loop for an existing plan file, checkpointing after implementation and capping fix rounds. Does not run spec-creator, implementation-planner, or test-writer."
when_to_use: "When you already have a plan file (e.g. plans/NN-slug.md, written by hand or by implementation-planner, run separately) and want implementer + the plan-verifier gate + a review fix loop in one command, instead of invoking each agent by hand and re-running the reviewer after every fix. NOT for turning an idea into a spec (spec-creator) or a spec into a plan (implementation-planner) — run those yourself first. NOT a replacement for your project's pre-push checks — run those separately, right before push. Does not write tests — that's test-writer, invoked separately, by design."
argument-hint: "<plan-path> [--auto] [--reviewer <agent>]"
user-invocable: true
---

# Implement Plan

One command over the pipeline's build-and-check agents:
`implementer → plan-verifier (gate) → reviewer (fix loop)`.

Arguments: `$ARGUMENTS` — a plan path, optionally with `--auto` and/or
`--reviewer <agent>`.

Agent names: installed as the `shvab` plugin, the agents are namespaced
(`shvab:implementer`, `shvab:plan-verifier`). Use the agent type exactly as
your `Agent` tool lists it.

Design rationale: [README.md](README.md).

**Deliberately out of scope:**

- `spec-creator` and `implementation-planner` — run these yourself, by hand,
  before invoking this skill. This skill starts from a plan that already
  exists; it never turns an idea into a spec or a spec into a plan.
- `test-writer` — run it yourself, by hand, once the implementation and
  review have settled. See [README.md](README.md) for how to bring it into
  this skill if you want it automatic.
- Your project's pre-push checks (full test suite, lint, typecheck, any
  pre-PR review) and any docs pass — still separate, deliberate steps after
  this skill finishes; it points you at them rather than running them.

## Step 0 — get the plan

Require a plan file path (conventionally `plans/NN-slug.md`). If you weren't
given one, stop and ask for it — do not infer a plan from a vague request or
from reading source code; that's `implementation-planner`'s job, not this
skill's. Confirm the file exists (`Read`) before proceeding.

State the plan path and, from its own "Modules affected" section, which
modules this run will touch, before doing anything else.

## The two policies this run follows

**Checkpoint after implementation, by default.** Once Phase 1
(`implementer`) finishes, stop, show a short summary of files changed, and
ask via `AskUserQuestion`: continue to the gate, stop here, or adjust
(feed a correction back to the same agent via `SendMessage`). Skip this
checkpoint only when the arguments above include `--auto`.

**Conservative auto-fix in Phase 3.** Only a finding the reviewer marks both
`CRITICAL`/`HIGH` **and** `CONFIRMED` triggers an automatic fix round through
`implementer`, capped at 2 rounds. `PLAUSIBLE` findings and anything `MEDIUM`
are reported, never auto-fixed — ask the user via `AskUserQuestion` whether
to fix, accept as known debt, or ignore each one still open after the
rounds. A reviewer that doesn't grade confidence at all gets no auto-fix:
every finding goes to the user.

## Relaying blocking stops

`implementer` stops mid-run and reports rather than guessing when: the plan
is ambiguous on something it needs, or it discovers the plan needs something
outside its stated scope. `plan-verifier` stops if either the plan path or
the implementation reference is missing — shouldn't happen here since Step 0
already confirmed the plan, but treat it the same way if it does. Neither
agent has `AskUserQuestion` in its own tool list — a stop can only appear in
the agent's report text.

When an agent's report is a stop rather than a finished result:

1. Extract the exact question or gap as the agent phrased it.
2. Ask the user via `AskUserQuestion`.
3. **Continue the same agent instance** (`SendMessage` to it) with the
   answer, instead of starting a fresh one — a fresh call throws away the
   context that agent already gathered.
4. Repeat until the report is a finished artifact or verdict, not a further
   stop.

This applies regardless of `--auto` — these are the agents' own contracts,
not this skill's checkpoint.

## Phase 1 — implementer

Invoke `implementer` with the plan path from Step 0. A mid-run scope-gap
report is a hard stop regardless of `--auto` — ask the user: tell
`implementer` to proceed with an explicit scope note, or stop the run here
and let the user go fix the plan themselves (via `implementation-planner`,
run by hand — out of this skill's scope).

When it finishes, read the report: files changed by module, and its own
test/typecheck self-check results — this is `implementer`'s own check, not
the pipeline's gate; that's Phase 2.

Checkpoint (unless `--auto`): show files changed, ask to continue.

## Phase 2 — plan-verifier (the gate)

Invoke `plan-verifier` with the plan path and "working tree" (or the branch,
if the user named one) as the implementation reference.

- `PASS` → proceed to Phase 3.
- `INCOMPLETE` → fix loop, capped at 2 rounds: dispatch `implementer` again
  with the plan path plus the specific Gaps list from the report, phrased as
  what to complete — the same plan, named gaps, not a new one. Re-run
  `plan-verifier`. Still `INCOMPLETE` after 2 rounds → stop and hand the
  user the Gaps table. Do not proceed to Phase 3 against code the gate
  hasn't passed.

This phase always runs, `--auto` or not — it's the deterministic gate the
rest of the pipeline depends on, not an extra checkpoint this skill adds.
`plan-verifier` currently runs on `sonnet`, which has no deterministic
backstop of its own — if its verdicts start looking wrong in practice
(missed gaps a manual look catches later), say so; that's a signal to move
it back to `opus` in the plugin's `agents/plan-verifier.md`, not something
this skill can self-correct.

## Phase 3 — review pass, with fix loop

**Pick the reviewer**, in this order: the agent named by `--reviewer`; else an
available agent that reviews architecture (typically `architecture-reviewer`,
from the project's `.claude/agents/` or another plugin — check your `Agent`
tool's list); else none. With no reviewer, skip this phase, say so in the
wrap-up, and point the user at their project's own review step (or
`/code-review`). Never invent a reviewer and never improvise the review
yourself.

Invoke it against the diff `plan-verifier` just passed.

Apply the fix-loop policy: collect every `CONFIRMED` `CRITICAL`/`HIGH`
finding into a fix list; dispatch `implementer` once per round with the plan
path plus that list, phrased as "additionally address these review
findings, staying strictly scoped to them" — pass this as plan *text*
alongside the plan path (`implementer` accepts plan text inline per its own
contract, so this doesn't require writing a new plan file). Re-run the
reviewer after each round. Cap at 2 rounds.

After the cap — or immediately, for `PLAUSIBLE`/`MEDIUM` findings — surface
whatever remains via `AskUserQuestion`: fix manually, accept as known debt
(note: this skill never edits a review tool's baseline or allowlist file —
that's a deliberate, separate act, not a side effect of "accept"), or ignore
for this run.

## Phase 4 — wrap-up

Report, not a checkpoint:

- Plan path.
- Files changed, by module.
- `plan-verifier`'s final verdict.
- The reviewer's final state: clean, or accepted findings and who accepted
  them — or "no reviewer available, phase skipped".
- Explicit next steps, none of which this skill ran:
  - **`test-writer`** — no coverage was added by this run; invoke it
    yourself when ready.
  - **Your project's pre-push checks** — the actual gate before opening a PR:
    full test suite, lint, typecheck, and any pre-PR review you use.
  - **A docs pass** — only if the feature is documentable and shipped.

## What this skill must not do

- Never invoke `spec-creator`, `implementation-planner`, or `test-writer` —
  out of scope by design, not an oversight.
- Never infer a plan from a vague request — Step 0 requires an actual plan
  file path.
- Never fabricate an agent's report — every phase transition is gated on
  that agent actually returning a finished artifact or verdict, never on an
  assumption of what it probably said.
- Never skip Phase 2, regardless of `--auto`.
- Never treat this skill's completion as equivalent to your project's
  pre-push checks, or as having added test coverage — say both explicitly in
  Phase 4.
