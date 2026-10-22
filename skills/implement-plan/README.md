# implement-plan — design rationale

The build-and-check part of the agent pipeline, run in one command:
`implementer → plan-verifier (gate) → reviewer (fix loop)`.

It starts from a plan that already exists. The earlier stages (`spec-creator`,
`implementation-planner`) and the later one (`test-writer`) stay manual, for
independent reasons:

## Why spec-creator and implementation-planner are out

Both are run by hand, by design — not a temporary cut. The judgment each does
(clarifying scope, choosing single- vs multi-agent execution) is exactly the
kind of decision the agents' own contracts treat as a mandatory human
checkpoint (`implementation-planner`: "never skip this question, even when
your recommendation feels obvious"). Folding them into an orchestrator that
also checkpoints risked producing two nested checkpoint experiences instead
of one clean one. Keeping them manual means: by the time this skill is
invoked, the plan already reflects a decision the user made deliberately, not
one relayed through this skill's own checkpoint UI.

## Why test-writer is out

Cost. `test-writer` runs a full test suite at least once and can trigger its
own fix round if it surfaces a suspected bug — real tokens, spent
automatically on every run. This is reversible: to bring it in, add a Phase 5
that invokes `test-writer` against the stable diff after Phase 3 settles,
with the same fix-loop shape capped at 2 rounds — probably behind an explicit
`--with-tests` flag rather than in the default path, so the cost-conscious
default doesn't regress silently.

## Why the gate agents default to a cheaper model

`plan-verifier` ships on `sonnet` and the reviewer is whatever agent your
project supplies. A gate can run 2-3 times per feature (once per fix-loop
round), so a top-tier model on every gate multiplies cost fast. The risk from
the downgrade differs by agent:

- **A reviewer backed by a deterministic check** (for example an
  architecture-rule linter whose result is independent of model quality) —
  low risk. The model's job is mostly to interpret that result and handle the
  judgment-only findings, which such an agent should label as judgment, not a
  machine result.
- **`plan-verifier`** — no deterministic backstop of its own. Its entire
  output (`DONE`/`PARTIAL`/`MISSING`/`CONTRADICTED` per plan item) is model
  judgment matching claims to `file:line` evidence, and it's the one gate the
  rest of the pipeline trusts before spending more tokens. A wrong verdict
  here defeats the reason it runs first. Treat `sonnet` as an experiment, not
  a settled call: watch its first several real verdicts, and move it back to
  `opus` in `agents/plan-verifier.md` if it starts producing a `PASS` that a
  later manual look or the reviewer contradicts.

## Why Phase 2 (plan-verifier) always runs, even with `--auto`

It's the one deterministic PASS/INCOMPLETE gate in the pipeline
(`plan-verifier`'s own design: "a closed gate, not advice"). `--auto` removes
this skill's *own* checkpoint after Phase 1 — it was never meant to remove a
gate the underlying agent is itself built to be.

## Why the reviewer is pluggable

The review step depends on what your project has: an architecture reviewer
agent in `.claude/agents/`, one from another plugin, or none. The skill
resolves it as `--reviewer <agent>`, then an available `architecture-reviewer`,
then skips the phase and says so. It never improvises a review of its own —
an unlabelled, unrepeatable review in the middle of an automated run is worse
than a clearly reported "no reviewer".

## Why the fix loop is conservative by default

The principle: a gate that fires on everything gets bypassed. Auto-fixing
low-confidence (`PLAUSIBLE`) or low-severity (`MEDIUM`) findings spends
`implementer` rounds on things that may be style, not defects — capping
auto-fix to `CONFIRMED` `CRITICAL`/`HIGH` keeps every automatic action
trustworthy, and keeps token spend concentrated on findings worth fixing
without a human looking first. A reviewer that doesn't grade confidence gets
no auto-fix at all.

## Fix-list dispatch uses plan text, not a new plan file

`implementer`'s own contract accepts a plan file path or the plan text itself
as input. Passing a fix list as inline text alongside the original plan path
keeps every fix-loop dispatch inside `implementer`'s actual contract, without
minting a new numbered file in the plans folder for what is really an
addendum to the same plan.

## Open questions

- **Unproven.** Treat the first few real runs as validating the design,
  including the model choices above, not just the feature they happen to
  build.
- **`sonnet` on `plan-verifier` is the riskiest single choice** — see above.
  If it needs reverting, that's a one-line frontmatter change in
  `agents/plan-verifier.md`, not a change to this skill.
- **`SendMessage`-based agent continuation** for relaying blocking stops
  hasn't been exercised in anger by this skill. Fallback: re-invoke the agent
  fresh with the original input plus the user's answer folded into the prompt
  text.
- **Fix-loop cap (2 rounds)** is a starting guess, not a measured number —
  revisit once real runs show whether findings typically converge in 1-2
  rounds or routinely need more.
