---
name: yourturn
version: 1.3.0
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

Read the relay config at `${user_config.config_path}` if it exists; its tables drive Steps 4 and 7.
Test for it — do not infer it from the absence of one. If it does not exist, say so as an
observation. If you have not looked yet, the config is *unchecked*, which is not the same claim.

**Write the scope down before you go any further.** This ritual scans exactly one repo, and every
check that cannot fire here — an adapter switched off, a missing config, another repo you also work
in but did not name — produces no output at all. A report that stays silent about them is
indistinguishable from one that looked and found nothing. Note, before Step 2:

- the absolute path you are about to scan;
- adapters marked `no` in the config; or that the config is absent, if you checked and it was;
  or that the config is unchecked, if you have not looked. Never widen either of those into a
  claim about paths you did not test — "no config in this repo" is reportable, "no config
  anywhere" is not;
- any check skipped because the files it needs are not in this repo.

Those belong at the top of the output as a scope line, not buried at the end. Do **not** go hunting
for the other repos to close the gap — naming what you did not look at is the point, and searching
the workspace is still forbidden by the target rule above.

### Step 2 — Sync

```bash
git branch --show-current
git status --short --branch
RELAY_BEFORE=$(git rev-parse HEAD)          # anchor: before the pull
git pull --ff-only; pull_rc=$?
RELAY_AFTER=$(git rev-parse HEAD)           # anchor: after the pull
git status --short --branch
printf 'RELAY_BEFORE=%s\nRELAY_AFTER=%s\npull exit=%s\n' "$RELAY_BEFORE" "$RELAY_AFTER" "$pull_rc"
```

Fall back to a plain `git pull` only if `--ff-only` is unavailable.

Steps 2.1 and 2.2 use those two anchors, never `ORIG_HEAD`. If the shell does not carry variables
across calls, substitute the SHAs you actually measured — do not reach for `ORIG_HEAD` instead.

**On conflict, divergence or any other error: stop.** Report what failed and which files are
affected. Do not resolve, reset, stash or check anything out. Do not continue to the later steps —
a pickup summary built on a failed sync is worse than no summary.

Stopping here means most of this ritual never ran. The scope line still goes out, but it may only
carry what you actually established before the sync failed — anything you had not reached yet is
*not checked*, never *not there*. A confident negative invented on the way out of a failure is the
worst thing this report can contain: it arrives exactly when the reader most needs to know what
was really looked at.

### Step 2.1 — What came in since you last picked up here

`git log -1` hides an entire night of work from the other machine. And "what this pull brought in"
is not enough either: if another session — or a plain `git pull` — already pulled, this pull
brings in nothing, and the other machine's work shrinks to "0 new commits".

So start from the last time `/yourturn` ran **on this machine**. That anchor lives in a per-machine
cache, one file per repo, never in git (a shared file would just be overwritten by the other
machine). Run this in the same shell as Step 2, or substitute the SHAs it printed:

```bash
ANCHOR_FILE="${XDG_CACHE_HOME:-$HOME/.cache}/relay-kit/last-relay-$(git rev-parse --show-toplevel | LC_ALL=C sed 's|[^A-Za-z0-9._-]|_|g')"
last=""; when=""
[ -f "$ANCHOR_FILE" ] && IFS=$'\t' read -r last when < "$ANCHOR_FILE"
if [ -z "$last" ]; then
  echo "no previous pickup recorded on this machine -> falling back to this pull"
elif ! git merge-base --is-ancestor "$last" "$RELAY_AFTER" 2>/dev/null; then
  echo "previous pickup ${last:0:8} ($when) is no longer in history (rebase/reset?) -> falling back to this pull"
  last=""
fi
if [ -n "$last" ]; then
  # commits made on this machine enter the reflog one by one; pulled ones only move the branch tip
  mine=$(git log -g --all --format='%H%x09%gs' 2>/dev/null \
    | awk -F'\t' '$2 ~ /^(commit|cherry-pick)/ || $2 ~ /Merge made by/ || $2 ~ /\((pick|squash|fixup|reword|edit)\)/ { print $1 }')
  ext=0; loc=0
  for h in $(git rev-list "$last..$RELAY_AFTER"); do
    if printf '%s\n' "$mine" | grep -Fqx "$h"; then loc=$((loc+1))
    else ext=$((ext+1)); git log -1 --format='%h %s' "$h"; fi
  done
  echo "since last pickup here ($when, ${last:0:8}): $ext from elsewhere (listed above), $loc made on this machine"
else
  git log "$RELAY_BEFORE..$RELAY_AFTER" --oneline   # fallback: what this pull brought in
fi
```

- Commits **from elsewhere** → list them all, no cap; this is the only place the other machine's
  work shows up. Head the section "since your last pickup here (<time>)".
- Commits **made on this machine** → the count only. You were there; `git log` has the details.
- Fallback → say why (first run here, or the anchor was rewritten away), then "**N new commits
  pulled**" and list them all. Never present a fallback as "since your last pickup".
- Nothing from elsewhere → one line, and move on.
- Group by the project prefix in the commit subject when there is one.
- Known blind spot: deleting a worktree branch deletes its reflog, so its commits count as "from
  elsewhere". That errs toward listing too much, never too little.

### Step 2.2 — Did this pull change the relay setup itself?

Instructions are loaded *before* the pull. If this pull updated the relay config or the skills
themselves, you are still running the old ones and the output can be quietly wrong — the worst
kind of failure, because nothing looks broken.

```bash
git diff --name-only "$RELAY_BEFORE..$RELAY_AFTER" | grep -E '(relay\.config\.md|skills/)' \
  && echo "relay setup changed in this pull — this run used the previous version"
```

If it fires, flag it on its own line at the top of the output and mark the affected sections as
possibly stale. **Do not silently re-run** — tell the user and let them decide. When it fires,
also skip recording the anchor in Step 3, so a re-run still sees the same commits.

### Step 3 — Platform and host, then record the anchor

```bash
uname -s
hostname
# only after a successful sync, and only if Step 2.2 did not fire
mkdir -p "$(dirname "$ANCHOR_FILE")" \
  && printf '%s\t%s\n' "$RELAY_AFTER" "$(date '+%Y-%m-%d %H:%M %z')" > "$ANCHOR_FILE" \
  && echo "recorded pickup anchor ${RELAY_AFTER:0:8}"
```

If the platform cannot be determined, report the hostname and carry on. If the anchor cannot be
written, say so — the next run will fall back to "what this pull brought in". This one-line file is
the only thing `/yourturn` writes, and it sits outside the repo.

The timestamp uses the machine's own clock and offset on purpose: setting `TZ=` to a zone name
fails silently on systems without tzdata (Git Bash on Windows, slim containers) and falls back to UTC.

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

**New since you left** (since your last pickup here, <time>: N from elsewhere, M made here — or the fallback reason and N pulled)
- <commits from elsewhere, all of them, grouped by project>

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
