# Journal frontmatter — edge cases

The compressed rules live in `SKILL.md`. Read this only when a case is genuinely ambiguous.

## Why frontmatter at all

`/yourturn` reads the newest journal entry to answer "what happened while I was away". Reading
the frontmatter costs a few dozen tokens; reading the body costs hundreds and buries the answer
in prose. Everything below exists to keep that read cheap and reliable.

## `tldr`

The one sentence you would want to read first if you came back after a week away.

Good:

```yaml
tldr: Fixed the filter flicker on web and shipped prompt v2.3
tldr: Late session on the laptop — cherry-picked the v2 branch, installed five skills
```

Bad — a list wearing a sentence's clothes:

```yaml
tldr: Did some work on the API, also looked at the frontend, and read some docs about caching
```

## `todos` vs `blockers`

The test is whether *you* can move it forward.

- **todo** — you know what to do, you just have not done it. "Write the migration test."
- **blocker** — something outside the work is in the way. "Staging DB credentials expired,
  waiting on ops."

An item that is both goes in `blockers`, because that is the list someone else may need to see.

## Pointing at projects instead of copying them

Wrong — the entry now has to be maintained:

```yaml
todos:
  - api: finish the retry test
  - api: rename the config key
  - api: update the changelog
  - web: fix the flicker
```

Right — the project's own plan stays the source of truth:

```yaml
todos:
  - api: 3 items open → see api/PLAN.md
  - web: fix the filter flicker
  - book the annual health check
```

## `next_hint`

One line, written for whoever sits down next — often you, on a different machine, having
forgotten everything.

```yaml
next_hint: Start the API first; the web dev server proxies to it and fails confusingly otherwise
next_hint: Pull before anything else — the desktop pushed three commits after I stopped
```

Omit the key when there is no real signal. An invented hint is worse than none: it will be
followed.

## `device`

Whatever tells you *which machine* — a hostname, or a short label like `laptop` / `desktop`.
It matters when two machines disagree about what is committed and you need to know who wrote
the entry.
