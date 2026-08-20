# Adapter: queue-check

**Used by:** `/afk`, `/yourturn`
**Turn it on if:** something runs unattended against this repo — a scheduled agent, a nightly job,
a bot that opens PRs from a queue file.

## Why

Unattended work lands where none of the other checks look: on branches that were never merged, in
PRs nobody opened, or nowhere at all because the queue is jammed.

The dangerous case is not "a PR is waiting". It is **a queue that stopped moving**: a finished item
left unticked makes a fail-closed runner skip the whole night, every night, silently. Every task
you add after that never runs, and nothing tells you.

## Configure

In your relay config's **Notes**:

```
queue-check: queue file = ops/night-queue.md, branch prefix = night/
```

## Check

```bash
Q=<queue file>
# first unchecked item
slug=$(awk '/^- \[ \]/{sub(/^- \[ \][[:space:]]*/,"");print;exit}' "$Q")

# has it already been done? (requires gh, and network)
gh pr list --state all --search "head:<prefix>" --limit 100 \
   --json headRefName,state,number --jq '.[] | .headRefName + " " + .state'
git ls-remote --heads origin '<prefix>*'
```

If the first queued item already appears in a remote branch or PR, the queue is **jammed**: the
runner keeps seeing it as done and refuses to move on.

## Report

- **Jammed** → put it first, above everything else in the output. It is a silent failure, and
  every later task is blocked behind it. Say what to remove.
- **Open PRs from the prefix** → one line each, plus the reminder that merging is not enough — the
  queue entry has to be ticked off too, since the runner treats the queue as read-only.
- **Neither, and both queries succeeded** → say nothing.
- **A query failed** (not logged in, offline, timed out) → say the check did not complete.
  Never let "could not find out" print as "nothing there". That is the one failure mode this
  adapter exists to prevent, and it applies to the adapter itself.

## Portability note

`timeout` does not exist by default on macOS. Guard it, or the command not found gets swallowed
by a redirect and the whole check silently reports "nothing found" — which looks exactly like
success.
