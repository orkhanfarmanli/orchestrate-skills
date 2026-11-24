---
name: flagman
description: Use when the user wants to go over a flags.md one entry at a time and decide what gets fixed, invoked as /flagman [path/to/flags.md]
argument-hint: "[path/to/flags.md]"
disable-model-invocation: true
---

# Go over a flags file

Arguments: `$ARGUMENTS`, the path to a `flags.md`, relative to the working directory. With no path, ask for one.

You lead the walk and never implement a fix. `/fixer` does that later from `fixes.md`. The only files you edit are the flags file, `fixes.md`, the spec and the tickets.

## Setup

Read the whole file. Its entries are the bullets under each heading, apart from `## Fix rounds`, which is a log, and apart from bullets that start `[fixed]`, `[refused]` or `[agreed]`, which are already settled. Each heading is `## NN <ticket title>`, and that ticket is `NN-*.md` in the issues directory beside the file.

Wherever this skill deletes an entry, delete its heading too once nothing is left under it.

Check that every path the entries cite still exists. An entry that cites files, all of them gone, is stale. When there are stale entries, the first reply is only their list, grouped under the commit that removed their files (`git log --diff-filter=D --oneline -1 -- <path>`) with one line per entry, and one question asking whether to delete them all, with the options `Delete all` and `Keep them`. On a yes delete them and start the walk in the same reply. Otherwise they join the points.

The remaining entries are the points. Entries count as one point only when the file says they are the same decision or one repeats the other. Count the points as M.

## One point per turn

Take the points top to bottom. For each:

1. Open what the entry cites: the lines at HEAD, and the ticket or screenshot when the entry leans on one. Where the cited code survives but the code that used it may not, add one grep for its use. That is the whole check. It confirms the entry still holds and tells you whether the product's users would notice. Cited code that is gone means that part of the entry no longer holds, so stop there instead of tracing where the behaviour moved. A render, a measurement or outside research waits until the user asks for it.
2. Reply in exactly this shape, then ask for the decision. Only the user's answer moves the walk forward.

```text
**Summary (N/M):** <one or two sentences>
**Recommendation:** <Fix or Ignore, then one or two sentences>
```

The summary is written for someone who has not read the ticket. It names the ticket, says what a user of the product would see, puts the entry's jargon in plain words, and carries a `file:line` to open.

The recommendation opens with `Fix` or `Ignore`. Recommend a fix when users would notice the problem or something breaks, and say what changes and what it costs. Otherwise recommend ignoring it and give the reason. An entry that no longer holds at HEAD gets `Ignore` with that as the reason. When the entry is a step only the user can take, such as a dashboard setting, give the step.

The tickets and the `spec.md` beside the flags file are the plan. The plan was written before the code existed and planning makes mistakes, so it tells you what was intended, and whether that was right is part of every point. Hold the plan to the same test as the code, which is whether users would notice a problem or something breaks. Where the two disagree, say which one is the mistake. Where they agree and the test still fails, both are, and the fix changes both. A fix the plan rules out is as open as any other.

Whichever way a point is decided, the plan should end up describing the product. An `Ignore` therefore rewrites a plan line the code contradicts, and a `Fix` rewrites a plan line the agreed fix contradicts.

## Asking

Put every question to the user through `AskUserQuestion`, one per call. A harness without that tool ends its turn on the question instead.

A point's question names the ticket and the point in a few words and ends `fix or ignore?`. Its header is `N/M`. Its options are always `Fix` first, described by the fix the user would be agreeing to, then `Ignore`, described as marking the entry refused. Each description also names the plan line that choice rewrites, when there is one. Append ` (Recommended)` to the label of the one you recommend. The dialog adds its own free-text choice, which is where a question, pushback or a different fix arrives.

## The answer

- **Ignore, drop, leave it:** keep the entry in the file and start its bullet with `[refused]`.
- **Fix:** keep the entry in the file and start its bullet with `[agreed]`. Copy the entry into `fixes.md` beside the flags file, under the same heading, and end the copy with `**Agreed fix:** <the fix>`. A different fix typed by the user is the agreed fix.
- **A question or pushback:** answer it in a sentence or two and ask again. It is not a decision.

After a decision, rewrite the plan line the chosen option named, when there is one, and present the next point in the same reply. A rewrite keeps the part it replaces in brackets as `[was: <old part>]`, with the new part straight after it.

## End of the walk

When the last point is decided, the user stops the walk, or Setup finds no points, end the turn on one line, `K fixes in <fixes path>. Run /fixer <fixes path> to implement them.` K counts the entries in `fixes.md`. With none, say that no fixes are waiting.
