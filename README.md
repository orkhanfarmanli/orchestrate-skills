# orchestrate-skills

Three Claude Code skills that take a directory of tickets to reviewed commits, then walk you through whatever still needs a human decision.

| Skill | Invoked as | What it does |
| --- | --- | --- |
| [orchestrate](skills/orchestrate/SKILL.md) | `/orchestrate <issues-dir> [model] [--parallel[=N]]` | Runs one fresh subagent per ticket on the current branch, reviews the whole run, runs fix rounds, and writes what needs you to `flags.md`. Never pushes or opens a PR. |
| [flagman](skills/flagman/SKILL.md) | `/flagman <path/to/flags.md>` | Walks `flags.md` one entry per turn. You answer fix or ignore, and agreed fixes are copied to `fixes.md`. |
| [fixer](skills/fixer/SKILL.md) | `/fixer <path/to/fixes.md> [model]` | Implements the agreed fixes in `fixes.md` with fresh subagents. Commits nothing. |

## Prerequisites

### Matt Pocock's skills (required)

This flow is built on top of [mattpocock/skills](https://github.com/mattpocock/skills) and does not run without them. `/orchestrate` reaches them in three ways, so all three have to be in place.

1. The Claude Code plugin. Ticket agents call `mattpocock-skills:tdd` and `mattpocock-skills:code-review` by their plugin names.

   ```bash
   claude plugin install mattpocock-skills@claude-plugins-official
   ```

2. The `implement` skill as a file. Ticket agents read it from `~/.agents/skills/implement/SKILL.md`, because a subagent cannot expand `/implement`.

   ```bash
   npx skills@latest add mattpocock/skills -g -s implement
   ls ~/.agents/skills/implement/SKILL.md
   ```

3. Tickets in his local-markdown layout. Run `/setup-matt-pocock-skills` once per repo and pick local files as the issue tracker, then produce the spec with `/to-spec` and the tickets with `/to-tickets`. That gives `/orchestrate` what it expects:

   ```text
   .scratch/<feature-slug>/spec.md
   .scratch/<feature-slug>/issues/<NN>-<slug>.md    **Status:** ready-for-agent, **Blocked by:** ...
   ```

### Also required

- The [superpowers](https://github.com/obra/superpowers) plugin. Fix-round agents call `superpowers:receiving-code-review`.
- Claude Code's built-in `code-review` skill, which the correctness reviews call at `xhigh`.

## Install

```bash
npx skills@latest add orkhanfarmanli/orchestrate-skills -g
```

Or copy the three directories under `skills/` into `~/.claude/skills/`.

## Flow

```text
/to-spec                                      writes spec.md            (Matt Pocock)
/to-tickets                                   writes issues/NN-slug.md  (Matt Pocock)
/orchestrate .scratch/<feature-slug>/issues   implements, reviews, writes flags.md
/flagman .scratch/<feature-slug>/flags.md     you decide each flag, writes fixes.md
/fixer .scratch/<feature-slug>/fixes.md       implements the agreed fixes, uncommitted
```

`/orchestrate` refuses to start on a dirty working tree, because ticket agents commit. `--parallel` runs up to N tickets at once in git worktrees and is described in [PARALLEL.md](skills/orchestrate/PARALLEL.md).
