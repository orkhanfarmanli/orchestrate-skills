# Parallel mode

**Siblings** are tickets in flight at the same time. Each sibling works in its own git worktree on its own branch. You **integrate** finished branches onto the current branch one at a time, from the main checkout. Everything in SKILL.md holds unless this file replaces it.

## Setup additions

Run these after SKILL.md's Setup steps. Any step that can't be completed downgrades the run to sequential. Say which step failed and why in your first message.

1. **Test lock.** Find `lockf` (macOS) or `flock` (Linux) with `command -v`. The lock file is `$(git rev-parse --git-common-dir)/orchestrate-tests.lock`. The **lock prefix** is `lockf <lock file>` or `flock <lock file>`. Every test command in the run carries it, whether you run it or a sibling does, so only one suite at a time touches the shared test database and cache.
2. **Worktree root.** Worktrees live in `<repo root>/.claude/worktrees/`, inside the repository, so a container that mounts the repository also sees them. If `git check-ignore -q .claude/worktrees/` fails, append `.claude/worktrees/` to `.git/info/exclude`.
3. **Worktree test command.** From the repository's AGENTS.md and CLAUDE.md, work out the command that runs one focused test file from inside a worktree. It must run the worktree's code against the shared test services, and it must include the lock prefix. Two gotchas:
   - A container wrapper launched from the worktree can start a second stack. Sail, for example, names the Compose project after the directory. Target the already running container with a working directory inside the mount instead.
   - Git inside a container can't follow a worktree's `.git` pointer to its host path, so git-dependent tools such as `pint --dirty` run on the host.
4. **Prove it** in the first worktree you bootstrap, before spawning anything: run one existing focused test file with the worktree test command and get it to pass. That same command goes into every sibling's prompt.

## Worktree bootstrap

Do this for each ticket, right after claiming it:

1. `git worktree add -b orchestrate/<feature dir name>/<NN> <worktree root>/orchestrate-<NN> HEAD`. Record the sha as the ticket's **base**. It includes every ticket integrated so far, so an unblocked ticket always starts on top of its blockers.
2. Copy in the gitignored config files that the test and lint commands read, for example `.env` or a local `phpunit.xml`. Copy config only, never caches.
3. Clone the dependency directories (`vendor`, `node_modules`) with copy-on-write: `cp -cR` on macOS, `cp -R --reflink=auto` on Linux. They must be real copies. Composer's autoloader resolves symlinks to their target, so a symlinked `vendor` loads the main checkout's classes.

## Loop

This replaces SKILL.md's one-at-a-time rule and its per-ticket steps 3 to 5.

- **Fill slots.** Take workable tickets in number order, up to N in flight. Claim each one and bootstrap its worktree, then spawn all the new agents in one message. Refill after each integration.
- **Check each report.**
  - In the worktree: `git -C <worktree> status --porcelain` is empty, and `git -C <worktree> log --oneline <base>..HEAD` shows at least one commit.
  - In the main checkout: `git status --porcelain -- . ":(exclude)<feature dir>"` is still empty. This is where a sibling that wrote outside its worktree shows up.
- **Integrate**, one ticket at a time:
  1. From the main checkout, run `git cherry-pick <base>..orchestrate/<feature dir name>/<NN>`.
  2. Rerun the report's test commands from the main checkout, under the lock prefix.
  3. If they pass: set `**Status:** done`, record flags as SKILL.md describes, then run `git worktree remove <worktree>` and `git branch -D <branch>`.
- **Fail.** Any of these counts as a failure:
  - the report says failed, or the agent errored out
  - a dirty tree, or no commit
  - a cherry-pick conflict, in which case run `git cherry-pick --abort`
  - tests go red after integrating

  On failure:
  1. Leave the ticket `claimed`.
  2. Write its flag entry with what stopped it and the worktree path. Keep the worktree and branch for inspection.
  3. Spawn nothing new. Let the siblings in flight finish, and integrate the ones that pass.
  4. Then give the final report. If tests went red after integrating, the commits stay on the branch. Say so, and leave them unreverted for the human.
- **Finish.** Once no workable ticket remains and nothing is in flight, rerun every integrated ticket's test commands once from the main checkout, under the lock prefix. Siblings that pass alone can still break each other. A red result goes in the final report, and the run counts as unclean. SKILL.md's Final review runs only after a green Finish, from the main checkout.

The final report adds the path of every worktree that was kept.

## Subagent prompt addition

Append this to every parallel agent's prompt, after the template's last paragraph, with every slot filled:

```text
You are running in parallel with sibling ticket agents currently in flight: <NN and H1 title of each sibling, or "none yet; more may start">. Siblings start and finish while you work and their commits land on the main branch; the orchestrator integrates your branch after you report, so you never rebase or merge.

Your worktree is `<worktree path>` on branch `<branch>`. Start every Bash command with `cd <worktree path> &&`, use absolute paths under the worktree for every read and edit, and commit there. The ticket and spec stay at the main-checkout paths above: tick boxes in your own ticket file there. A wrong spec line becomes a flag instead of an edit, because siblings are reading the spec; this overrides the spec-editing line above. Keep every change inside your worktree and your own ticket file.

The test database and cache are shared with siblings. Run every test as `<worktree test command>` with a 10-minute Bash timeout: its lock waits while a sibling's suite runs. Each run rebuilds the shared test database from the running worktree's migrations, so a sibling's schema appearing or vanishing between your runs is expected. Run lint and format commands inside your worktree.
```
