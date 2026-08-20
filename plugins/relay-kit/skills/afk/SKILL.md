---
name: afk
version: 1.0.0
description: |
  Read-only end-of-session sweep: what did I leave behind before walking away from this machine?
  Triggers on "afk", "logging off", "done for today", "packing up", "switching to my other machine".
  Never commits, pushes or stops anything on its own — it reports, the user decides.
user-invocable: true
tags:
  - workflow
  - sync
  - cross-device
---

# AFK

The first of the three relay rituals: **`/afk` leaves the machine clean**, `/handoff` writes down
what only exists in the conversation, `/yourturn` picks the work back up.

## Why this exists

Broken handoffs almost never start at the arrival end. They start at the departure end — the
other machine pulls, finds nothing, and only then discovers the work was committed but never
pushed, or a service is still holding a port, or the one file that explained the state was never
written. By then you are somewhere else.

`/afk` is the checklist you run *before* that.

## Core principles

- **Read-only scan, then a list.** It tells you what is still loose. You decide what to do.
- **Never** commit, push, stash, reset, restore files, stop services or delete anything on its own.
- Any check that fails (a broken repo, a command the platform lacks) is marked "unverified" and
  the sweep continues. One failed check must not swallow the others.

## Steps

### Step 0 — Platform, host, repo root

```bash
hostname
uname -s                                   # Darwin on macOS, MINGW*/MSYS* on Git Bash
REPO_ROOT=$(git rev-parse --show-toplevel 2>/dev/null)
```

If you are inside a git worktree rather than the main checkout, say so — the sweep still runs,
but the user should know which tree they are looking at:

```bash
[ "$(git rev-parse --git-dir)" != "$(git rev-parse --git-common-dir)" ] && echo "worktree session"
```

If `REPO_ROOT` is empty, fall back to the configured `repo_root` (`${user_config.repo_root}`).
If that is empty too, report the host and stop guessing.

Read the relay config at `${user_config.config_path}` if it exists. Its **Projects**, **Services**
and **Adapters** tables drive the rest of this sweep. If it does not exist, run the git checks
anyway and mention once that a config would make the sweep project-aware.

### Step 1 — Main repo sweep

```bash
git -C "$REPO_ROOT" status --short --branch
git -C "$REPO_ROOT" log @{u}..HEAD --oneline 2>/dev/null   # committed but not pushed
```

Report in this order of severity:

- **Committed but not pushed** — highest priority. This is the single most common way a handoff
  breaks: the work exists, and the other machine cannot see it.
- **Uncommitted changes** — list the files. Anything that is notes, docs or agent-facing state
  ranks above code here: those are exactly the files the next session reads first.
- **Untracked files** — list them, but separate "new files that should be tracked" from
  "junk that was never going to be committed" (lock files, local databases, build output).
  Do not nag about the second kind.
- **A branch with no upstream** — remind the user that pushing needs `-u` the first time.

### Step 2 — Drift on the projects you touched

Only check projects this session actually touched. Do not sweep the whole repo; that just makes
noise the user learns to skip.

Touched projects come from the union of: paths in `git status --short`, paths in
`git diff --name-only @{u}..HEAD`, and anything you know you edited this session.

For each one, look up its row in the relay config and compare timestamps:

```bash
git -C "$REPO_ROOT" log -1 --format=%ct -- <project>
git -C "$REPO_ROOT" log -1 --format=%ct -- <first-read path>
```

- Code or changelog newer than the first-read doc, or state changed this session while the entry
  document did not → **possible drift**.
- Both updated together → no drift.
- Not enough evidence → "unverified". Do not guess.

Drift is a **soft warning only**. `/afk` never edits the first-read document.

### Step 3 — Machine-local settings in a shared file

If `.claude/settings.json` is tracked by git, machine-local preferences written into it by the
UI will follow you to the other machine and cause a conflict there.

```bash
git -C "$REPO_ROOT" ls-files --error-unmatch .claude/settings.json >/dev/null 2>&1 \
  && git -C "$REPO_ROOT" diff --name-only -- .claude/settings.json
```

If it shows up as modified, say so and offer the fix — move the machine-local keys to
`.claude/settings.local.json`, which is git-ignored and still takes precedence. **Only after the
user says yes.** No hit, no line in the output.

### Step 4 — Running services

Ports come from the **Services** table in the relay config. With no config, skip this step
rather than guessing at ports.

```bash
# macOS / Linux
lsof -nP -iTCP -sTCP:LISTEN 2>/dev/null | grep -E ":(<ports>)"
# Windows (Git Bash)
netstat -ano | grep -E ":(<ports>) "
```

Anything still listening: name the port and its project, and note that the other machine will
hit a port conflict or a locked database if it starts the same service. Nothing listening: one
line saying so.

### Step 5 — Projects missing from the config

New top-level directories that never made it into the relay config are how the *next* session
ends up reading the wrong file.

```bash
CFG="$REPO_ROOT/${user_config.config_path}"
[ -f "$CFG" ] && for d in "$REPO_ROOT"/*/; do
  name=$(basename "$d")
  case "$name" in .*|node_modules|dist|build|target|vendor) continue;; esac
  [ -z "$(ls -A "$d" 2>/dev/null)" ] && continue
  grep -q "$name" "$CFG" || echo "not in relay config: $name/"
done
```

Offer to add the missing rows. **Wait for a yes.** Nothing missing → say nothing.

### Step 6 — Adapters

Run the checks enabled in the config's **Adapters** table, reading each one's instructions from
the plugin's `adapters/<name>.md`. Adapters that are off produce no output at all — a check that
cannot fire should not take up a line.

### Step 7 — The list, and the follow-up questions

End with the checklist, then any offers to act. Every offer waits for a yes; that is what keeps
this skill read-only by default.

If the repo keeps a daily journal and today's entry does not exist yet, mention it once. Do not
insist — `/afk` is an operational sweep, journalling is reflection, and they are different jobs.

## Output

Under 35 lines. Only flag things that will actually make the next session stumble; untracked junk
is not one of them. If everything is clean, say so in one line instead of padding a checklist.

```markdown
**AFK sweep**
- Host: <hostname> | Repo: <path> <(worktree) if applicable>

**Loose ends**
- Committed but not pushed: <N> commits — `git push`
- Uncommitted: <files>
- <nested repo / adapter findings>

**Services still up**
- :5173 web — the other machine will collide on this port
- (or "nothing listening")

**Drift on projects you touched**
- `<project>` — first read: `<path>` | drift: none / possible / unverified
  - evidence: <what you compared>

**Other**
- Today's journal: written / not yet
- Relay config: all projects listed / `<name>/` missing
- <notes from the config's Notes section>

**Verdict**
- Clean, safe to walk away — or: <N> things to deal with first

**Next**
- On the other machine, run `/yourturn`
- Handing off to another tool or a fresh session? Run `/handoff` first
```
