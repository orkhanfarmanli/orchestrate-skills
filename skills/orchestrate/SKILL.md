---
name: orchestrate
description: Use when the user wants a directory of tickets implemented end to end by one fresh subagent per ticket, invoked as /orchestrate <issues-dir>
argument-hint: "[issues-dir] [model] [--parallel[=N]]"
disable-model-invocation: true
---

# Orchestrate a ticket set

Arguments: `$ARGUMENTS`. The first token is the issues directory. The second token, when present, is the model for the ticket agents; without it they run on the harness default. The feature directory is the issues directory's parent; `spec.md` and `flags.md` live there. You coordinate and verify. You never implement a ticket or a fix yourself.

`--parallel` or `--parallel=N`, anywhere in the arguments and not counted as a token above, switches to parallel mode with at most N tickets in flight (default 3). Without it the run is sequential. In parallel mode, read `PARALLEL.md` in this skill's directory before Setup: it adds worktree setup, replaces the one-at-a-time rule and per-ticket verification, and extends the subagent prompt.

## Setup

1. Resolve the issues directory to an absolute path and list it. Tickets are `NN-slug.md`. Read every ticket's `Status:` and `Blocked by:` lines.
2. From the repository root run `git status --porcelain -- . ":(exclude)<feature dir>"`. If it prints anything, stop and report: ticket agents commit, and would sweep those changes in. Note the current branch; all work lands there. No new branches, no push, no PR. Record `git rev-parse HEAD` as the run's **start**.
3. Create `flags.md` in the feature directory if missing, containing only a `# <feature directory name> flags` heading.

## Loop

A ticket is workable when its `Status:` is `ready-for-agent` and every ticket in `Blocked by:` is `done`. Re-read every ticket's `Status:` line before each pick and take the lowest-numbered workable ticket. In sequential mode, run exactly one ticket at a time: start the next only after the previous agent's report has arrived. Stop when no workable ticket remains.

Per ticket:

1. Set `**Status:** claimed` in the ticket file. Record `git rev-parse HEAD`.
2. Spawn one fresh agent with the Agent tool: `subagent_type: general-purpose`, `model` set only if the second token was given, prompt from the template below. The agent needs the Agent tool itself for its review step, so never pick a type that lacks it.
3. When its report arrives, verify from the repository root: `git status --porcelain -- . ":(exclude)<feature dir>"` is empty and `git log --oneline <recorded sha>..HEAD` shows at least one commit.
4. Pass: set `**Status:** done`. Append the report's flags that clear the bar below to `flags.md` under `## NN <ticket H1 title>`, and its `## Review findings` under a `### Review findings` subheading there.
5. Fail (report says failed, tree dirty, no commit, or the agent errored out): leave `claimed`, append `## NN <ticket H1 title>` to `flags.md` with one line saying what stopped, stop the loop, report.

## Flag bar

An entry earns its place only if a human must decide or do something: a spec or ticket contradicted by what the code actually does, a deploy step that breaks if followed as written, a destructive hazard for another machine, a decision left open. Bold one-line lead, then the file or command to check. Never: implementation choices visible in the code, restatements of ticket status, pre-existing failures unrelated to the branch, unrequested feature ideas, failures that announce themselves on first run, speculative scenarios. A wrong line in a ticket or spec is fixed in that file, not flagged.

## Correctness review

Call the Skill tool with skill `code-review` and args `xhigh`, followed by the target when one is given. This is the built-in review the user runs as `/review`, a different skill from `mattpocock-skills:code-review`. Wait for its findings before continuing. Check each finding against the code and drop the ones that don't hold. Fix only the important ones: wrong results, crashes, data loss, security holes, broken callers. Rerun the tests each fix touches. List every confirmed finding in the report's `## Review findings` section, one line each with its file:line, and start each one you fixed with `[fixed]`.

## Subagent prompt template

Fill every angle-bracket slot with absolute paths. Paste the Flag bar and Correctness review paragraphs verbatim where marked.

```text
Implement ticket <NN>: `<ticket path>`. The spec is `<spec path>`; read both before touching code.

Follow the implement skill: run `cat ~/.agents/skills/implement/SKILL.md` and do what it says. The Skill tool refuses that skill and a `/implement` line does not expand inside a subagent, so read the file. Its `/tdd` and `/code-review` are `mattpocock-skills:tdd` and `mattpocock-skills:code-review`, invoked through the Skill tool. Where the repository's AGENTS.md or CLAUDE.md disagrees with the implement skill, the repository wins.

After that review and before committing, `git add -N` any new files so the diff shows them, then run a correctness review of your uncommitted diff with no target. <Correctness review paragraph>

The acceptance boxes are the ceiling. Build the smallest diff that ticks them and stop: reach for the standard library, a platform feature, or a dependency already installed before writing new code, and keep a single implementation concrete rather than behind an interface, a factory, or a config key. Before committing, re-read your diff against the boxes and delete what no box needs. A decision the ticket leaves open is a flag, not something to settle in code.

Commit to the current branch. Do not create or switch branches, push, or open a PR. Tick the ticket's acceptance boxes as you meet them. Fix a wrong line in the ticket or spec in that file, and keep what it said. Put the part you replace in brackets as `[was: <old part>]` and write the corrected part straight after it. Do not edit other tickets.

End your report with exactly these sections: `## Result` (done or failed, one line), `## Commits` (`git log --oneline` of your commits), `## Verification` (commands run and their outcome), `## Review findings` (every confirmed finding of every review you ran, each one you fixed starting `[fixed]`, or "none"), `## Flags` (entries meeting this bar, or "none"). <Flag bar paragraph>
```

## Final review

Once the loop ends with no ticket failed and at least one ticket done this run, record `git rev-parse HEAD` as the run's **end** and spawn one more fresh agent the same way as a ticket agent, with this prompt:

```text
Commits `<start>..<end>` on the current branch implement tickets from `<issues dir>`. The spec is `<spec path>`.

First review the run against the whole spec: call the Skill tool with skill `mattpocock-skills:code-review` and args `fixed point <start>, spec <spec path>`. Check each finding against the code and drop the ones that don't hold. Fix a requirement that a ticket covered but left partial or wrong. A requirement no ticket covered, scope creep, and every Standards finding stay unfixed. All of them go into the report's `## Review findings` section, one line each with its file:line or spec line, the fixed ones starting `[fixed]`. Commit those fixes before the next review.

Then run a correctness review with the target `<start>..HEAD`. <Correctness review paragraph>

Commit to the current branch. Do not create or switch branches, push, or open a PR.

End your report with exactly these sections: `## Result` (done or failed, one line), `## Commits` (`git log --oneline` of your commits, or "none"), `## Verification` (commands run and their outcome), `## Review findings` (every confirmed finding of every review you ran, each one you fixed starting `[fixed]`, or "none").
```

When its report arrives, check that `git status --porcelain -- . ":(exclude)<feature dir>"` is empty. Append `## Final review` to `flags.md` with the report's review findings, or with one line saying what stopped if it failed or left the tree dirty.

## Fix rounds

Run these once the final review reports done with a clean tree. They leave `flags.md` holding what was fixed and what needs the user.

1. Collect every review finding in `flags.md`: the `### Review findings` entries and the `## Final review` lists. Bold flags are decisions and stay for the user. Entries that start `[fixed]`, `[refused]` or `[agreed]` are settled, by an earlier round or by the user in a `/flagman` walk, and stay as they are. Group the findings by the code they cite, one group per app or feature area, each finding in exactly one group.
2. Take one group at a time. Record `git rev-parse HEAD`, then spawn a fresh agent the same way as a ticket agent, with the prompt below.
3. When its report arrives, verify from the repository root: `git status --porcelain -- . ":(exclude)<feature dir>"` is empty, `git diff <recorded sha>..HEAD` changes only what the report's `## Fixed` list describes, and the report's test commands pass when you rerun them.
4. Pass: start each of the report's fixed findings with `[fixed]` in `flags.md`. Delete its dropped findings, and any heading left with nothing under it. Append each kept finding's reason to its entry, so the entry says what is unclear or what the user must decide. Fail (report says failed, the agent errored out, a dirty tree, a change outside `## Fixed`, or red tests): leave the group's findings, run no more rounds, and leave its commits for the user.

Then append `## Fix rounds` to `flags.md` with one line per round run: the group, its commit shas, and its fixed, dropped and kept counts, or what stopped it.

```text
Fix the review findings below, left on the current branch by automated reviews of the tickets in `<issues dir>`. The spec is `<spec path>`. Each finding cites file:line and sits under the heading of the ticket that produced it; line numbers may have drifted since.

<the group's findings, each copied exactly, under its ticket heading>

Call the Skill tool with skill `superpowers:receiving-code-review` and follow it, treating these findings as an external reviewer's. Nobody can answer questions mid-run, so where it says to ask or to stop and discuss, keep that finding for the user. An unclear finding holds back only itself, so go on fixing the rest.

Sort every finding into the first of these that fits:
- **Fix** when it holds at HEAD and has one obvious fix that follows from the spec, its ticket, or a written rule in AGENTS.md or CLAUDE.md, stays within a few files, and involves no product choice.
- **Drop** when it no longer holds at HEAD, repeats another finding, or isn't worth a change. Not worth a change means cosmetic, a race on rare writes that loses no data, reachable only from a stale client, on a path nothing calls yet, or a departure from a convention that its sibling code shares.
- **Keep** for the user in every other case, such as a product or spec decision, a fix with more than one sensible shape, a change that sprawls, anything you're unsure of, or a fix you tried and couldn't get working. Back out a failed attempt before moving on.

Make each fix the smallest diff that resolves its finding, and give every behavior change a test that fails without the fix. Run the full test suite and the lint and type checks of every app you touched. Commit to the current branch. Do not create or switch branches, push, or open a PR.

End your report with exactly these sections: `## Result` (done or failed, one line), `## Commits` (`git log --oneline` of your commits, or "none"), `## Verification` (commands run and their outcome), then `## Fixed`, `## Dropped` and `## Kept`. Those three list findings copied exactly, one per line, or "none". Give each Dropped finding a short reason, and each Kept finding what is unclear or what the user must decide, plus what you tried if a fix failed. Every finding above appears in exactly one of them.
```

## Final report

A table of tickets with status and commit shas, the final review's and fix rounds' commits, the `flags.md` path with its count of open entries, and tickets left unworked with the reason. Ask the user to review `flags.md` once now that the set is done.
