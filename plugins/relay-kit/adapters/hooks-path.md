# Adapter: hooks-path

**Used by:** `/yourturn`
**Turn it on if:** your repo ships git hooks in a versioned directory (`.githooks/`, `scripts/hooks/`)
that every machine is supposed to be using.

## Why

`.git/hooks/` is per-clone and never travels with a pull. A hook everyone agreed on keeps working
on the machine where it was installed and silently does nothing everywhere else — which is exactly
where the mistake it was meant to catch will happen.

`core.hooksPath` points git at the versioned directory instead, so pulling the hook is enough to
have it.

## Detect

```bash
git config core.hooksPath
```

Empty, or not the repo's hooks directory, means this machine is not running the shared hooks.

## Fix

```bash
git config core.hooksPath <hooks-dir>
```

This writes to `.git/config` — local to this clone, not a change to the repo — so it is safe to do
without asking, unlike anything that touches tracked files. Say you did it, in one line.

Hooks must be tracked as executable (`100755`). Git Bash on Windows honours the shebang and does
not need the executable bit, but macOS and Linux do.

## Verify

Do not assume it took effect:

```bash
git config core.hooksPath          # reads back the value
ls -l <hooks-dir>                  # confirms the hook exists and is executable
```

Once set, git ignores `.git/hooks/` entirely. Old copies there are harmless but no longer run —
worth saying once, so nobody debugs the wrong file.
