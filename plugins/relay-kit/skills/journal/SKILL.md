---
name: journal
version: 1.0.0
description: |
  Guided daily learning journal — seven questions, written to a dated Markdown file with
  frontmatter the relay rituals can read cheaply.
  Triggers on "journal", "write my journal", "log today", "that's a wrap for today".
  Also used to remind the user of yesterday's unfinished items at the start of a new day.
user-invocable: true
tags:
  - journal
  - reflection
  - productivity
---

# Journal

Seven questions in the order **facts → feeling → reflection → improvement → action**, written to
one file per day.

Journalling is optional in this kit, but it feeds the other rituals: `/yourturn` reads the
frontmatter of the most recent entry to answer "what happened while I was away", and reads only
the frontmatter, so it stays cheap.

> The question set is adapted from a daily-journal skill by Raymond Hou.

## Where entries go

- Vault root: `${user_config.vault_root}` — if empty, the repo root is used instead.
- Journal directory: `${user_config.journal_dir}`, relative to whichever of those applies.
- Full path: `<root>/<journal_dir>/YYYY-MM/YYYY-MM-DD.md`.

Create the month directory if it does not exist.

## Writing an entry

### Step 0 — Date, and what already exists

1. Run `date` — never assume today's date.
2. If today's file already exists, ask whether to **add to it** or **rewrite it**. Adding means
   reading the current entry and merging; rewriting replaces it.
3. Read yesterday's entry if there is one. If its "tomorrow" list has items, mention them before
   starting:

   > Yesterday you wanted to: <items>. Worth checking off while we do this.

### Step 1 — Ask, one question at a time

Wait for each answer before asking the next. "Skip" means skip.

1. **What did you do today? What happened?** — anything counts: installed, tried, changed, read.
2. **Anything that surprised or delighted you?** — a tool doing something unexpected, a concept
   finally clicking.
3. **What got you stuck? How did you get out?** — including things still unsolved. Problems are
   worth more here than wins.
4. **What is unfinished and carries over?**
5. **What went well? How did the day feel overall?**
6. **One small thing you control that you could do better tomorrow?**
7. **Top three for tomorrow?**

### Step 2 — Write the file

The frontmatter is what the relay rituals read, so it comes first and it is not optional:

```yaml
---
date: YYYY-MM-DD
device: <hostname or short machine label>
tldr: one sentence, under 20 words, the single most important thing from Q1
todos:            # from Q4 + Q7, at most 5, one line each
  - ...
blockers:         # from the unsolved parts of Q3. Omit the whole key if none
  - ...
next_hint: ...    # one line: the first thing to do next session. Omit the key if unclear
---
```

Frontmatter rules:

- `tldr` is for a human scanning in two seconds. Detail belongs in the body.
- `todos` holds high-priority items only, not everything from Q4 and Q7.
- **Point at projects, do not copy them.** A project with its own plan gets one pointer line
  (`api: acceptance pending → see api/PLAN.md`). Only one-off items get expanded here. Copying
  project state into a daily entry creates a second source of truth that starts rotting the next
  morning.
- Omit `blockers` and `next_hint` entirely when empty. Never leave empty keys.

See `references/frontmatter.md` for edge cases.

Then the body — skip any section the user skipped rather than leaving an empty heading:

```markdown
# YYYY-MM-DD

## What happened
## What got stuck        (problem -> fix or current status)
## How it felt
## Surprises
## What went well
## One small improvement
## Carrying over
- [ ] ...
```

Keep the user's own words. Do not summarise their answers into something tidier than they said.
Mark anything an AI session did on their behalf with an `[AI]` prefix so the record stays honest
about who did what.

### Step 3 — Confirm, then save

Show the entry and ask whether anything should change before saving.

### Step 4 — Offer to commit

Only if the journal lives inside a git repo, and only as an offer:
`journal: YYYY-MM-DD`.

## Next-day reminder

When a new session starts and yesterday's entry has unfinished items, mention them once. Read the
frontmatter `todos` first; only fall back to scanning the body if there is no frontmatter.

Include the "one small improvement" line if there was one — that one is easy to forget and it is
the only part of the entry that is about how the work gets done rather than what got done.

## Notes

- Ask like a friend, not like a form. Do not chase short answers with follow-up questions.
- Never pressure the user into writing. A skipped day is fine.
- The entry is private. It does not need to be well written.
