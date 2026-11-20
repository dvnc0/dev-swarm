# /swarm

A Claude Code command plus a worker agent. `/swarm <feature>` turns the main session into a manager: it plans the feature, splits it into small tasks, runs a swarm of cheap worker subagents in parallel git worktrees, merges the results, verifies, and reports. Faster and cheaper than one big agent doing everything.

Model split:
- **Manager:** Opus (the command, `model: opus`). Plans, dispatches, merges, verifies. Writes no feature code.
- **Workers:** `sw-worker` agent, Haiku for mechanical well-specified edits, Sonnet (medium effort) for anything needing judgment. The manager picks the tier per task.
- **Recon:** built-in `Explore` agent (Haiku), only when needed.

## How it works

0. **Preflight.** Git repo, clean tree, base branch + HEAD recorded, test/build command detected, `sw-worker` shadow check.
1. **Recon.** Light: `Explore` sweeps or a few direct reads.
2. **Plan.** Writes `spec.md` (goal, scope, design, contracts, acceptance) and `tasks.md` (id, owned files, wave, tier, acceptance check, context pointers). No file is owned by two tasks in a wave; max 10 workers per wave.
3. **Approval gate.** Shows a task table and waits for go-ahead (skipped with `--yes`).
4. **Execute waves.** Creates `swarm/<slug>`, one worktree per task under `.git/swarm-worktrees/`, spawns the wave's workers in one parallel message. Workers commit on their own branch.
5. **Merge.** The manager merges each task branch `--no-ff` into `swarm/<slug>`, resolves conflicts, removes the worktree and branch, sanity-checks after each wave.
6. **Verify.** Full test/build on `swarm/<slug>`, acceptance criteria checked, failures become fix tasks (max 2 rounds).
7. **Final merge + report.** Fast-forwards (else `--no-ff`) into the base branch and deletes `swarm/<slug>`. On failure it leaves the branch intact. Writes `report.md`.

A worker that reports blocked/failed is retried once (clarified prompt, or Haiku bumped to Sonnet), then the run stops.

## Requirements

- Claude Code (recent; `permissionMode` and `effort` frontmatter).
- Run inside a git repo with a clean working tree, on a branch (not detached HEAD).
- A test or build command is strongly recommended.

## Install (symlinks)

User scope (recommended). Run from this repo's root. Check that each symlink target doesn't already exist first.

```bash
SRC="$PWD"
mkdir -p ~/.claude/{commands,agents}
ln -sn "$SRC/commands/swarm.md" ~/.claude/commands/swarm.md
ln -sn "$SRC/agents/sw-worker.md" ~/.claude/agents/sw-worker.md
```

Per-project variant, for a repo that should carry the toolkit (run from that repo's root, with `SRC` set to this repo's path):

```bash
mkdir -p .claude/{commands,agents}
ln -sn "$SRC/commands/swarm.md" .claude/commands/swarm.md
ln -sn "$SRC/agents/sw-worker.md" .claude/agents/sw-worker.md
```

The shadow check accepts the per-project variant: a `.claude/agents/sw-worker.md` symlink is OK when `readlink -f` resolves it to a file whose `../commands/swarm.md` exists in the same toolkit. A regular file or a symlink into a different tree counts as a shadow.

Uninstall:

```bash
rm ~/.claude/commands/swarm.md ~/.claude/agents/sw-worker.md   # only if they are symlinks to this repo
```

The symlinks break if this repo is moved; re-create them.

## Usage

```
/swarm add a --json output flag to the CLI and document it
/swarm --yes add rate limiting middleware to the API with tests
/swarm migrate the date helpers from moment to date-fns
```

Headless:

```bash
claude -p "/swarm --yes add a /health endpoint with tests" --permission-mode auto < /dev/null
```

## Caution

Name shadowing: a project-level agent whose frontmatter `name:` is `sw-worker` overrides the installed one, and a trusted repo's `.claude/` is trusted code. As a best-effort check, Phase 0 runs `grep -RlE 'sw-worker' --include='*.md' <dir>/.claude/agents` over the session's working directory, the target repo root, and every `--add-dir` directory, and stops on any match that isn't a symlink into this toolkit (with `--yes` it warns loudly and continues) (observed: agents are matched by `name:` not filename, the folder is scanned recursively, symlinks are followed, project agents override user agents). It can't catch every way a repo can change agent behavior. Don't run `/swarm` with `--yes` in a repo you don't trust: workers run with `permissionMode: auto` and execute shell commands, including the repo's own tests.

## Artifacts

`<repo>/.context/swarm/<slug>/`: `spec.md`, `tasks.md`, `report.md`. `.context/` is git-ignored, or added to `.git/info/exclude` if it isn't, so the tree stays clean. Worktrees live in `.git/swarm-worktrees/<slug>/<id>` and are removed after merge.

Branches: integration `swarm/<slug>`; tasks `swarm-task/<slug>/<id>`, deleted after merge with `git branch -d` (re-runs and fix tasks get new ids like `t3-fix1`; an aborted task branch is kept for the report) (git can't hold both `swarm/<slug>` and `swarm/<slug>/<id>`).

## Cost notes

- Opus planning and merge overhead dominates small features (in one 5-task test run Opus was about 92% of cost). The swarm pays off on larger, more parallel features: use `/swarm` when there are 4 or more meaningful tasks, otherwise use a single agent.
- Opus only plans, merges, and reads capped command output. Reading and writing code is delegated.
- Haiku is used wherever a task is mechanical and fully specified; Sonnet elsewhere.
- Each worker pays its own context start-up, so tiny tasks cost more than they save; the planner uses one worker for 1-2 file features.
- Retries and fix rounds (max 2) add cost. Use the approval gate to review the plan before spending.

## Limitations

- Needs a clean tree and a branch; won't run otherwise.
- Worktrees hold tracked files only: no `node_modules`, venvs, or build output. Workers install what their check needs, which can be slow or fail offline.
- Parallel speedup depends on the plan: a deep dependency chain means many serial waves.
- Merge conflicts mean the planner mis-assigned ownership; the manager resolves or reruns the task.
- Tasks that need services, secrets, or network may not be verifiable by workers.
- No plugin manifest on purpose: plugin agents ignore `permissionMode`, which `sw-worker` needs.
