---
name: yourturn
version: 1.2.0
description: |
  Pick up work on a machine you have been away from: sync, then report what changed, where the
  work stands, and what still needs starting.
  Triggers on "yourturn", "your turn", "I'm back", "picking this up", "what did I miss".
  Never resets, stashes, force-pulls or resolves conflicts on its own.
user-invocable: true
tags:
  - workflow
  - sync
  - cross-device
---

# YourTurn

The third of the three relay rituals: `/afk` leaves the machine clean, `/handoff` writes down what
only exists in the conversation, **`/yourturn` picks the work back up.**

This is not `git pull` with extra steps. Pulling tells you the files changed. This tells you
**what the work is now, and what you have to do about it** — which projects moved, which have open
work, what state has gone stale, and what services need starting on this machine.

## Core principle

Sync first, then interpret. If the sync fails, stop — never produce a summary that *looks* like a
successful pickup when the tree is not actually up to date.

## Steps

### Step 1 — Target and scope

Resolve the target the way `project-sync` does: an explicit path if given, otherwise the current
working directory if it is a git repo. Never search the workspace by project name, never fall back
to a default project, never change directory on the user's behalf.

Read the relay config at `${user_config.config_path}` if present; its tables drive Steps 4 and 7.

**Write the scope down before you go any further.** This ritual scans exactly one repo, and every
check that cannot fire here — an adapter switched off, a missing config, another repo you also work
in but did not name — produces no output at all. A report that stays silent about them is
indistinguishable from one that looked and found nothing. Note, before Step 2:

- the absolute path you are about to scan;
- adapters marked `no` in the config, or the fact that there is no config;
- any check skipped because the files it needs are not in this repo.

Those belong at the top of the output as a scope line, not buried at the end. Do **not** go hunting
for the other repos to close the gap — naming what you did not look at is the point, and searching
the workspace is still forbidden by the target rule above.

### Step 2 — Sync

```bash
git branch --show-current
git status --short --branch
git pull --ff-only
git status --short --branch
```

Fall back to a plain `git pull` only if `--ff-only` is unavailable.

**On conflict, divergence or any other error: stop.** Report what failed and which files are
affected. Do not resolve, reset, stash or check anything out. Do not continue to the later steps —
a pickup summary built on a failed sync is worse than no summary.

### Step 2.1 — What actually came in

`git log -1` hides an entire night of work from the other machine. After pulling:

```bash
git log ORIG_HEAD..HEAD --oneline 2>/dev/null    # what this pull brought in
git log --since="2 days ago" --oneline -15       # recent activity either way
```

- Commits in `ORIG_HEAD..HEAD` → say "**N new commits pulled**" and list them all.
- Five or fewer recent commits → list them. More → list five and say how many remain.
- Nothing new → one line, and move on.
- Group by the project prefix in the commit subject when there is one.

### Step 2.2 — Did this pull change the relay setup itself?

Instructions are loaded *before* the pull. If this pull updated the relay config or the skills
themselves, you are still running the old ones and the output can be quietly wrong — the worst
kind of failure, because nothing looks broken.

```bash
git diff --name-only ORIG_HEAD..HEAD 2>/dev/null | grep -E '(relay\.config\.md|skills/)' \
  && echo "relay setup changed in this pull — this run used the previous version"
```

If it fires, flag it on its own line at the top of the output and mark the affected sections as
possibly stale. **Do not silently re-run** — tell the user and let them decide.

### Step 3 — Platform and host

```bash
uname -s
hostname
```

If the platform cannot be determined, report the hostname and carry on.

### Step 4 — Where the work stands

**If the user named a project:**

1. Find its row in the relay config. If there is no config or no row, fall back to the project's
   `PLAN.md`, then `README.md`, and say which one you used.
2. Read that first-read document and record the path you actually read.
3. Report five fixed fields: **Current**, **Next**, **Blocked**, **Last verified**, **Staleness**.
4. For `Staleness`, compare what the document claims against git history and recent changes.
   Newer code with an untouched entry document means the document is probably behind. Not enough
   evidence means "unverified" — never guess.

**If the user named no project**, do not pick one. Give a lightweight index instead: for every
project in the config (or every `PLAN.md` one level down if there is no config), count the
unchecked checkboxes in its first-read document.

```bash
for f in $(find . -maxdepth 2 -name PLAN.md -not -path ./PLAN.md 2>/dev/null | sort); do
  proj=$(dirname "$f" | sed 's|^\./||')
  boxes=$(grep -cE '^[[:space:]]*([-*]|[0-9]+[.)])[[:space:]]*\[ \]' "$f")
  [ "$boxes" -gt 0 ] && echo "$proj: $boxes open"
done
```

Report one line per project with open items. **State the limit honestly**: this counts checkboxes
in one file per project. Projects that track work in prose, or in a separate progress file, are
undercounted — say so rather than implying the index is complete.

### Step 5 — Reconcile branch hints

If anything you read (a handoff note, a journal entry, a plan) claims the work sits on some
branch, verify it before repeating it:

```bash
git merge-base --is-ancestor <branch> main && echo "already merged into main"
git diff --stat main..<branch>
```

**What git says beats what the note says, always.** A stale branch hint sends the next hour of
work to the wrong place.

### Step 6 — Environment on this machine

Things that are per-machine and therefore never arrive with a pull:

- `.env` / `.env.local` present?
- Virtualenv present (`Scripts/activate` on Windows, `bin/activate` elsewhere)?
- `node_modules` present, or lockfile newer than it?
- Browser runtimes (Playwright and friends) installed?
- Ports from the config's **Services** table free, or already taken?
- Anything in the config's **Notes** worth repeating here.

### Step 7 — What to start (always run this)

This is the last step and therefore the easiest one to skip. Do not skip it.

List every service in the config's **Services** table with its port and start command. If there is
no config, look for what the repo actually offers:

```bash
find . -maxdepth 3 -name launch.json -path '*/.claude/*' -not -path '*/worktrees/*' 2>/dev/null
```

If nothing turns up, **say so explicitly** ("no services configured") rather than omitting the
section — an empty section is indistinguishable from a skipped step.

### Step 8 — Adapters

Run the checks enabled in the config's **Adapters** table, following each one's instructions in
the plugin's `adapters/<name>.md`. Disabled adapters produce no output of their own — they
are named in the scope line instead (Step 1), so "switched off" never reads as "checked, clean".

## Output

Under 40 lines, scannable in one pass. Mark anything uncertain as "unverified" rather than
smoothing it over.

```markdown
**YourTurn — picked up**
- Host: <hostname> | Repo: <path> | Branch: <branch>
- Scope: this repo only — `<absolute path>`
- Not checked: <adapters off / no relay config / other repos you work in> ← omit when nothing was skipped
- Pull: <result> | Working tree: <clean / N modified>

**New since you left**
- <commits, grouped by project>

**Project state** (when a project was named)
- First read: `<path actually read>`
- Current: <where it stands>
- Next: <up to 5>
- Blocked: <or "nothing">
- Last verified: <what, when — or "unverified">
- Staleness: fresh / possibly behind (+ evidence) / unverified

**Open work** (when no project was named — one line each)
- `<project>` — <N> open items (details: `/yourturn <project>`)

**This machine**
- .env: <ok / missing> | deps: <ok / reinstall> | ports: <free / taken>

**Start when you need them**
- `npm run dev` — web (5173)
- `uvicorn app.main:app --reload` — api (8000)

**Suggested first move**
- <one thing>
```

## Rules

- No reset, no checkout to discard changes, no automatic stash.
- No automatic commit, push, or stopping services.
- Never resolve a conflict on the user's behalf.
- Never overwrite uncommitted work.
- A missing journal or config is not an error — note it and continue.
- If the sync fails, stop. Do not produce a summary that reads like a successful pickup.
