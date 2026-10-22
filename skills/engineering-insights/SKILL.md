---
name: engineering-insights
description: "Append a non-obvious engineering lesson to the touched module's LEARNINGS.md (the nearest one above the changed files; created at the module root if the project has none). Reads the file first and extends an existing entry instead of duplicating it. Use at the end of a substantive session, or the moment something non-obvious surfaces mid-task."
when_to_use: "Finished a task that involved a real problem, fix, or discovery; hit a dead end worth recording; found surprising library, tooling, or framework behavior; or the user says 'log this', 'capture what we learned', 'update learnings', 'write it down'."
---

## Pick the file

The lesson goes in the `LEARNINGS.md` of the module you worked in:

1. The nearest `LEARNINGS.md` walking up from the files you changed — in a
   monorepo that's the package's own file; in a single-package repo, usually
   the root one.
2. None found, but `CLAUDE.md` or the project docs name a learnings or
   agent-memory file → use that file instead.
3. Still none → create `LEARNINGS.md` at the module root (the repo root for a
   single-package repo) with the seven headings listed under "Where to file",
   then add the entry. Say in your reply that you created the file.

A lesson that spans modules goes in the file of the module where the fix
lives; name the other modules' paths inside the entry.

## Before writing

**Read the module's `LEARNINGS.md` in full before writing.** If the lesson is
already there, extend that entry — never add a second copy.

## Editing rules

**Never overwrite.** Insert with `Edit`; never `Write` an existing file — that
replaces every existing entry (`Write` is only for creating a file that
doesn't exist yet). Anchor the edit on the section heading and put the new
entry directly under it, leaving the rest of the file byte-identical. Add one
entry per edit. Never reword, reformat, or delete an entry you didn't just
write — to correct an old one, add a new dated entry that supersedes it.

If a hand-written file lacks the heading you need, file under the closest
heading it does have, or add the missing standard heading at the end as its
own edit first.

## Where to file

File under the matching section: What Works · What Doesn't Work · Codebase
Patterns · Tool & Library Notes · Recurring Errors & Fixes · Session Notes ·
Open Questions.

## What qualifies

Record only what would change a future session's behavior: name the exact
file, function, or observed behavior, and date the entry. Skip general best
practice, one-off context, and anything obvious from reading the code. If
nothing clears that bar, write nothing and say so.

Level of specificity required: [examples.md](examples.md).
