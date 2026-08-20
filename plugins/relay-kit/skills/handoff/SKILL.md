---
name: handoff
version: 1.0.0
description: |
  Write a handoff note that another session, another model or another person can pick up cold.
  Triggers on "handoff", "hand this off", "write a handoff note", "I'm switching to <other tool>",
  "context is getting long, start a fresh session".
  Read-only apart from the note itself: never commits, pushes, stops services or edits project code.
user-invocable: true
tags:
  - workflow
  - handoff
  - context
---

# Handoff

The second of the three relay rituals: `/afk` leaves the machine clean, **`/handoff` writes down
what only exists in this conversation**, `/yourturn` picks the work back up.

## Why this exists

When you switch tools, run out of quota, or start a fresh session because context got long,
**whoever picks up cannot see this conversation.** They can only read the disk.

- **Durable layer** — git history, the project's first-read doc, your notes. Already on disk.
  Point at it. Do not copy it.
- **Volatile layer** — decisions you settled on, questions still open, approaches you already
  proved were dead ends. These live *only* in the conversation and evaporate with it.

The single job of this skill: **squeeze out the volatile layer, add pointers to the durable
layer, and write both into one self-starting note.**

## Core principles

- **Read-only except for the note.** No commit, no push, no stash, no stopping services, no
  code edits. Getting the work committed is `/afk`'s job and the user's call.
- **Pointer vs payload.** Durable things get a path. Volatile things get written out in full,
  because they exist nowhere else.
- **Honesty beats tidiness.** Unresolved problems and dead ends are the highest-value part of
  the note. A note that only lists wins makes the next session repeat your mistakes.
- **Redact before writing.** The note gets pasted into other tools. Strip keys, tokens,
  passwords and personal data first.

## Steps

### Step 0 — Locate the repo and the target project

```bash
REPO_ROOT=$(git rev-parse --show-toplevel 2>/dev/null)
```

If that is empty, fall back to the configured `repo_root` (`${user_config.repo_root}`).
If that is empty too, **ask** — do not guess a project.

**Target project**, which decides the note's content and filename:

1. The user named it ("hand the api work over to …") → use that.
2. Otherwise, whichever project the working directory sits in.
3. Still ambiguous → stop and ask. Never fall back to a default project.

**Purpose of the handoff**, only if the user said it ("handing off for QA", "fresh session to
finish the refactor"): record it and let it shape what you keep in Step 2 — a QA handoff wants
verify commands and known-weak areas, not implementation trivia. If they did not say, skip the
line entirely rather than inventing one.

### Step 1 — Read the durable layer (to make pointers, not copies)

```bash
git -C "$REPO_ROOT" log -8 --oneline
git -C "$REPO_ROOT" status --short --branch
git -C "$REPO_ROOT" log @{u}..HEAD --oneline 2>/dev/null   # committed but NOT pushed
```

- Read the relay config (`${user_config.config_path}`) and find the target project's row.
  Its **First read** becomes the note's canonical pointer — path only, never a copy of the
  contents. If the project has no row, fall back to its `PLAN.md`, then `README.md`, and say
  in the note which one you used.
- If the repo keeps agent-facing notes or gotchas, link the ones relevant to this project.
- If `git log @{u}..HEAD` returned anything, the note must say so **loudly**: those commits
  are invisible to whoever pulls. Push first, or the handoff is a lie.

### Step 2 — Squeeze the volatile layer

This is the part no tool can recover later. Go back over the session and pull out three lists,
one line per item:

- **Decisions** — settled calls the next session should not relitigate, each with its reason.
- **Unresolved** — 2 to 5 open questions.
- **Dead ends** — what you tried and why it failed.

If a list is genuinely empty, write "none this session". Do not pad.

### Step 3 — Redact

Scan everything about to be written. Replace secrets and personal data with
`<redacted: what it was>`, and add one line at the end of the note saying what was redacted and
where the real value lives.

### Step 4 — Write the note

```bash
mkdir -p "$REPO_ROOT/.claude/handoffs"
OUT="$REPO_ROOT/.claude/handoffs/$(date +%Y-%m-%d)_<project>_handoff.md"
```

Use the template below. **The first line must be the self-starting sentence** ("You are picking
up …") so that pasting the file is enough to start — the reader should not need to know any
keyword or command.

### Step 5 — Report, with two pasteable versions

Print the path, a short summary, and both forms from "Output" below:
a **pointer version** for tools that can read the repo, and a **self-contained version**
for tools that cannot. The self-contained one must already be redacted.

## Note template

```markdown
# Handoff: <project> — <YYYY-MM-DD>

You are picking up work on <project>. Follow the steps below, and **report your understanding
plus a plan before changing any code.**

## 1. Read first (durable state, all on disk)
1. `CLAUDE.md` / `AGENTS.md` — working agreements for this repo
2. Canonical pointer: `<first-read path>` — current state lives here; read it, don't duplicate it
3. `git log -8` and `git status` — recent commits and uncommitted work
4. <related notes or gotcha files, if any>

## 2. Volatile state from the last session (git cannot show you this)
- Goal / definition of done: <one line>
- Purpose of this handoff: <only if stated>
- Where it got to: <progress>
- Files touched: <list>
- Decisions: <each with its reason>
- Unresolved: <2-5 open questions>
- Dead ends: <approach -> why it failed>

## 3. How to pick up
- Verify command: `<from the relay config>` — expect: <success evidence>
- Services to start: <from the relay config, with ports>
- Unpushed commits: <shout, or "none">
- Report progress and a plan before editing code.

Redaction: <what was masked / where the real value lives>
```

## Output

Keep it under 20 lines.

```markdown
**Handoff note written**
- Project: <project>
- File: `.claude/handoffs/<YYYY-MM-DD>_<project>_handoff.md`
- Volatile layer: <N> decisions, <M> unresolved, <K> dead ends
- <unpushed warning, if any>

**How to use it**

A. Pointer version (the other tool can read this repo):

   Read .claude/handoffs/<file> and pick up from there.

B. Self-contained version (paste anywhere, no repo access needed):

```text
You are picking up <project>. Read, report your understanding, and give me a plan
before changing code.
[Read first, if you can reach the repo] CLAUDE.md, <first-read path>, git log -8, git status
[Goal / done] <one line>
[Progress] <where it got to> | Files touched: <list>
[Decisions] <each with its reason>
[Unresolved] <2-5>
[Dead ends] <approach -> why it failed>
[Verify] <command> -> expect <evidence>
[Unpushed] <warning or "none">
[Redaction] <what was masked / where the real value lives>
```
```

## Boundaries

- Never commit, push, stash, reset or stop services — that is `/afk` plus the user.
- Never edit project code, and never edit the first-read document. Read it, point at it.
- Do not stash a second copy of project state in a temp directory. One source of truth.
- Do not replace `/yourturn`. That skill is for the person arriving; this one is for the
  person leaving.

## Rules

- The only write is the note itself. Everything else is read-only.
- Any read that fails (missing file, unsupported git flag) is marked "unverified" and the note
  still gets written. Never block on it.
- Redact before writing, every time.
