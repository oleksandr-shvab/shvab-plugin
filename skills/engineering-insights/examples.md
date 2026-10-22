# What a LEARNINGS entry has to look like

The bar: **an agent reading this cold, with no memory of the session, knows
exactly what to do or avoid.** If knowing it wouldn't change what the agent
does, it isn't an entry.

## The test, in four questions

| | |
|---|---|
| **Specific** | Names a file, function, table, or observed behavior — not a topic. |
| **Reusable** | A *future* session benefits, not just the one that found it. |
| **Actionable** | It changes a decision. If it changes nothing, skip it. |
| **Dated** | `### YYYY-MM-DD — short title`, so staleness is visible later. |

## Not entries

- ❌ "Promises can be tricky" — a topic, not a lesson.
- ❌ "Be careful with async" — changes no decision.
- ❌ "The reviews module handles reviews" — obvious from the code.
- ❌ "Renamed a variable in `helpers.ts`" — one-off, no future value.
- ❌ "Always validate user input" — general best practice, not project knowledge.

## Entries

✅ **Names the exact failure and the exact remedy**

> `Promise.all()` on the ingest pipeline times out past 30 items — use
> `Promise.allSettled()` with batches of 10 for this module.

✅ **Names the file and says why the alternative fails**

> Checkout-flow state always goes through Zustand (`cartStore.ts`) — three
> components share the cart, so local state silently desyncs them.

## Worked examples by kind

The three kinds of entry that pay for themselves most. These are
*illustrative* — invented to show the level of detail. Your entries must name
your own real files, functions, and observed behavior.

✅ **A discovery that redirects the whole approach** — *What Works*

> The billing service already computes proration; the invoice job just
> discards it. `invoice-job.ts` destructures `const { lineItems, total } =
> result;`, dropping `result.prorationCents`. A refactor removed only the
> *rendering* of it, never the calculation — so showing proration on invoices
> is a template wiring job, not a new calculation in `ProrationCalculator`.
> Check what `BillingResult` already carries before touching the calculation
> path.

Why it works: names the file, the exact line, and the history, and ends with
a rule that redirects a future session before it wastes an hour.

✅ **A trap that type-checking won't catch** — *Recurring Errors & Fixes*

> Adding `deletedAt: z.date().nullable()` to the `Order` schema makes the
> *key required* — the value may be null, the key may not. Every object
> literal typed as `Order` stops compiling, including fixtures far from the
> change (`test/fixtures/orders.ts`, `scripts/seed.ts`). Use `.nullable()`
> where the key is always present; `.nullish()` where sibling fields are
> already optional-and-nullable.

Why it works: explains the *mechanism* (`nullable` ≠ optional key), lists
where the breakage surfaces, and gives the decision rule for next time.

✅ **A measurement that overturns the obvious choice** — *What Works*

> Real job durations here are 0.4–9 s (measured against `job_runs` in
> staging). A whole-seconds formatter — what `formatDuration` did with
> `Math.round(ms / 1000)` — renders every sub-second job as "0 s" or "1 s",
> so the fast majority is indistinguishable. Switch to milliseconds below 1 s
> and `toPrecision(2)` seconds above.

Why it works: carries the measured range, so a future session doesn't
re-derive it — and names the specific wrong answer someone would reach for.

## Adding an entry without clobbering the file

This file only ever grows. Existing entries are other sessions' work — some
of them were expensive to learn.

**1. Read the whole file first.** Not a slice of it. Editing from a partial
read is how content gets dropped, and it's also how duplicates get written:
the entry you're about to add may already be three sections down.

**2. Insert with `Edit`, never `Write`.** `Write` replaces the file wholesale
— one call and every prior entry is gone. `Edit` fails loudly when its anchor
doesn't match uniquely, which is exactly the safety you want here.

**3. Anchor on the section heading.** Each of the seven headings appears
exactly once, so it's an unambiguous anchor, and the edit touches only the
two lines around the insertion point:

```
old_string:   ## What Doesn't Work

new_string:   ## What Doesn't Work

              ### 2026-07-29 — <title>

              <entry>
```

Everything already under that heading shifts down untouched. Anchoring on an
existing *entry* instead risks swallowing it if the match is loose.

**4. One entry per edit.** Two insertions in one call means a bad anchor can
damage both.

**5. Correcting an old entry: supersede, don't rewrite.** Add a new dated
entry that says what changed and why. History stays readable, and a wrong
correction can't silently erase a right original. The only edit permitted to
an existing entry is appending to it — never rewording or reflowing it.

**6. Nothing to add is a valid outcome.** If the session produced no lesson
that clears the bar, say so and leave the file untouched. Padding it lowers
the signal for every future read.

## Placement

| Section | Holds |
|---|---|
| What Works | An approach that proved out, with the reason it did |
| What Doesn't Work | Dead ends and antipatterns — the most-skipped, most-valuable section |
| Codebase Patterns | Conventions and architecture decisions specific to this repo |
| Tool & Library Notes | Dependency quirks and surprising framework behavior |
| Recurring Errors & Fixes | An error seen more than once, plus its fix |
| Session Notes | Dated one-line summaries of substantive sessions |
| Open Questions | Left unresolved, worth picking up later |
