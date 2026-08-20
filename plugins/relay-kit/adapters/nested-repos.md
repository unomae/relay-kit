# Adapter: nested-repos

**Used by:** `/afk`, `/yourturn`
**Turn it on if:** your repo contains other git checkouts — a vendored project, a fork you work on
in place, a sibling repo someone cloned inside this one.

## Why

The outer repo's `git status` says nothing about a nested checkout. Work committed there is
invisible to every check in the main sweep, so it is the classic way a "clean" machine turns out
to have a day of unpushed work on it.

## Detect

```bash
find . -maxdepth 3 -name .git -not -path "./.git*" 2>/dev/null
```

For each hit:

```bash
git -C <nested> status --short --branch
git -C <nested> log @{u}..HEAD --oneline 2>/dev/null
```

## Report

- Uncommitted or unpushed work in a nested repo → list it at the same severity as the main repo.
  This is the whole point of the adapter.
- Clean → one line per repo, or a single line covering all of them.
- Every nested repo clean → say nothing at all.

## Rules

- Read-only, exactly like the ritual that called you. Never commit or push inside a nested repo,
  not even when the main repo is being committed.
- A nested repo that fails to respond (no upstream, broken checkout) is marked "unverified" and
  does not block the rest of the sweep.

## Note for test runners

Test runners often walk into nested checkouts and count their tests as yours. If a suite's test
count jumps for no reason, check whether a nested repo or worktree was added, and exclude it
explicitly rather than trusting the total.
