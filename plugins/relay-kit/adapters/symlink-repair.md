# Adapter: symlink-repair

**Used by:** `/yourturn`
**Turn it on if:** your repo tracks symlinks and anyone on the team works on Windows.

## Why

Git stores a symlink as a mode `120000` blob. On a Windows checkout without symlink support, git
writes a **plain file containing the link target** instead — mode `100644`. The checkout succeeds,
nothing warns you, and the next commit turns a tracked symlink into a one-line text file for
everyone else on the team.

The failure is silent in both directions, which is why it needs a check rather than a habit.

## Detect

```bash
git diff --raw | awk '$1 == ":120000" && $2 == "100644" { print $6 }'
```

Any output means symlinks have been flattened in the working tree. No output means nothing to do —
say nothing.

## Repair

Report the list first and get a yes. Then, per file:

```bash
git checkout -- <path>
```

If it flips straight back, the platform is not creating symlinks at all. On Windows that needs
either Developer Mode enabled or an elevated shell, plus:

```bash
git config core.symlinks true
```

Verify afterwards — re-run the detect command and confirm it is empty. A repair you did not
re-check is not a repair.

## Rules

- Never `git checkout --` anything without showing the list and getting a yes first: that command
  discards working-tree changes, and a file that is *both* flattened and legitimately edited would
  lose the edit.
- If the same file flips back a second time, stop and report the platform problem instead of
  looping.
