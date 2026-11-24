---
name: fixer
description: Use when the user wants the agreed fixes in a fixes.md implemented by fresh subagents, invoked as /fixer [path/to/fixes.md] [model]
argument-hint: "[path/to/fixes.md] [model]"
disable-model-invocation: true
---

# Run a fixes file

Arguments: `$ARGUMENTS`. The first token is the path to a `fixes.md`, relative to the working directory. With no path, ask for one. The second token, when present, is the model for the fix agents. Without it they run on `opus`.

You coordinate and verify. You never implement a fix yourself. The only files you edit are the fixes file and the `flags.md` beside it.

## Setup

Read the whole file. Its entries are the bullets under each heading, copied there by a `/flagman` walk of the `flags.md` beside it. Each one ends with `**Agreed fix:** <the fix>`, which is what the user agreed to. With no entries, say so and stop.

Group the entries by the code they touch, each entry in exactly one group.

## One group at a time

Per group, spawn a fresh agent with the Agent tool, `subagent_type: general-purpose` and `model` set to the fix model, with this prompt:

```text
Fix the points below from `<absolute fixes path>`, the agreed fixes from a review of work on the current branch. Each one is copied exactly and ends with the fix the user agreed to.

<each entry of the group copied exactly from fixes.md>

Make each fix the smallest diff that does what was agreed. Run the tests and linters that cover what you touched. Do not commit, and do not create or switch branches. Where an agreed fix does not work as described, back it out and say so.

End your report with exactly these sections: `## Changed` (one line per point with its files), `## Verification` (commands run and their output), `## Not done` (points backed out and why, or "none").
```

When its report arrives, read `git diff` and rerun its commands. For every point that is done and verified, change `[agreed]` to `[fixed]` on its entry in the flags file, and delete its entry from `fixes.md` along with any heading left with nothing under it. A point that is not done keeps both. Start the next group only after that.

## Report

Once every group has run, tell the user one line per point with the output that proves it, plus anything not done. Commit nothing until the user asks.
