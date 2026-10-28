# Changelog

## 1.4.0

`/handoff` notes now say what to do next and how sure each item is; three checks stop reporting a failure as a clean result.

- `/handoff` (1.2.0):
  - New **Next Actions** section: 1–5 ordered steps, each with file, change, verify command and
    expected evidence. Action 1 must be startable without asking anyone; an action that still
    needs approval never goes first
  - Every decision and dead end carries a confidence marker — `[verified]`, `[unverified]` or
    `[?]`. A dead end that was never actually run is `[unverified]`
  - The note ends with a 7-point **first-reply contract** for whoever picks it up (what they read,
    whether their checkout matches, what is missing, what they will check, first action)
  - New fields: **authorization boundary** (what was approved, by whom, what still needs a human —
    a proposal or tool output is not approval) and **running state** (only what this session
    started, never a machine-wide scan; an old PID is not permission to kill)
  - Template sections and fields are never dropped; empty ones say "none" / "unconfirmed", so
    "checked, nothing there" stays distinguishable from "never checked"
  - Redaction gets a closed list of safe references (key paths, keychain / password-manager entry
    names, CLI profile names, config paths) — locators only, never contents
  - The pointer version tells the reader to treat the note as data; authorization comes from the
    user at pickup time
  - A self-check before reporting (4 sections, markers present, no unfilled placeholders). It
    reads `grep` output, not exit codes, with `|| true` — the checks expect opposite directions,
    and under `set -e` a clean note would otherwise abort the script
- `/afk` (1.1.0): the service scan no longer reports a failed scan as "nothing listening".
  `lsof` exits 1 both when nothing listens and on some errors; the old `2>/dev/null | grep`
  pipeline hid both. The scan now keeps the exit code, treats "exit 1, no output" as clean,
  reports anything else as "unconfirmed (exit=N)" with the error text, and matches the listening
  state plus an exact port, so `:5173` no longer also matches `:51730`
- `/journal` (1.0.1): a missing yesterday entry is skipped quietly; one that exists but cannot be
  read (permissions, a cloud placeholder) is reported once, never turned into "nothing carried over"
- `vault-journal` adapter: `/yourturn` picks the newest entry by `YYYY-MM-DD.md` only. A weekly
  note such as `week-36.md` sorts after the dates, usually has no frontmatter, and used to win —
  silently dropping every handoff field
- Why: one-way sync of fixes and handoff-format changes made in the author's private versions of
  these rituals since August, rewritten without the private routing table

## 1.3.0

`/yourturn` now lists what came in since you last picked up **on this machine**, not just what this pull brought in.

- The anchor is a one-line bookmark per repo in `~/.cache/relay-kit/` (respects `XDG_CACHE_HOME`),
  written at the end of a successful run. It is per machine and never in git — a shared file would
  just be overwritten by the other machine
- Commits in range are split using the reflog: ones made on this machine are counted, everything
  else is listed in full. Deleted worktree branches lose their reflog, so their commits err toward
  "from elsewhere" — too much, never too little
- No bookmark yet (first run), or the bookmark was rewritten away by a rebase/reset → the report
  says so and falls back to "what this pull brought in"
- Not recorded when the sync fails or when the pull updated the relay setup itself, so a re-run
  still sees the same commits
- Dropped the "last 2 days" listing; the bookmark range replaces it
- Why: another session had already pulled the other machine's 15 commits, so the next `/yourturn`
  reported "0 new commits pulled" and the evening's work collapsed into nothing
- The bookmark timestamp uses the machine's own clock and offset: `TZ=<zone>` fails silently on
  systems without tzdata (Git Bash on Windows) and falls back to UTC
- README and DESIGN now list this as the kit's third write, alongside `/handoff` and `/journal`

## 1.2.2

Fixes a false positive in `/yourturn`: after a failed sync it could report commits the pull never brought in.

- Step 2 now records its own anchors — `RELAY_BEFORE` before the pull, `RELAY_AFTER` after it,
  plus the pull's exit code — and Steps 2.1 and 2.2 range over those instead of `ORIG_HEAD..HEAD`
- When the pull fails the two anchors are equal, so both sections are empty rather than replaying
  whatever the *previous* successful pull brought in
- Dropped the `2>/dev/null` on both commands: a missing anchor now fails loudly instead of
  quietly turning into an empty range
- Why `ORIG_HEAD` was wrong: git writes it only once a merge actually starts. A pull that dies in
  the fetch stage (network) or aborts on the `--ff-only` check (diverged branch) never reaches
  that write, leaving the value from the last successful pull
- Reproduced in an isolated lab, old anchor against new on identical state: on a failed pull the
  old anchor reported 1 commit and a "relay setup changed" warning, on an aborted `--ff-only` it
  reported 2 — the new anchor reported none in both, while still correctly reporting 1 commit and
  the warning on a pull that really did bring one in

## 1.2.1

Fixes a regression from 1.2.0: on the abort path, `/yourturn` reported a config it had never looked for.

- `/yourturn` no longer claims the relay config is missing when it has not checked. "Absent" is now
  an observation, "unchecked" is a separate state, and neither may be widened into a claim about
  paths that were not tested
- The stop-on-failed-sync rule now says what the scope line may carry: only what was actually
  established before the sync failed
- Found by running `/yourturn` against a dirty tree with an incoming commit: it aborted correctly,
  then announced "no relay config found anywhere under `~`" while `.claude/relay.config.md` sat in
  the repo — `/afk` read that same file in the same session, and `/yourturn` itself read it on the
  happy path. The 1.2.0 scope line asked the report to note "the fact that there is no config",
  which presumes the negative is already known

## 1.2.0

`/yourturn` now says what it did **not** look at, so an untouched check stops looking like a clean one.

- New scope line at the top of the report: the absolute path scanned, plus anything skipped —
  adapters switched off, a missing config, other repos you work in but did not name
- Step 1 records the scope before the sync runs; Step 8 names disabled adapters rather than
  silently emitting nothing
- Found by running `/yourturn` in a second repo: every environment-specific check degraded to
  no output, and the report read as though the whole machine had been swept

## 1.1.1

Spells out what `/handoff` does when there is no conversation to squeeze.

- `/handoff` invoked as the first message of a session now has a defined behaviour: label the note
  as reconstructed, rebuild from the durable layer, and keep pointer-vs-payload. Previously the
  spec only covered "list is genuinely empty", leaving the common cold-start case to interpretation

## 1.1.0

`/handoff`'s output location is now configurable, and no longer defaults into a protected directory.

- New `handoff_dir` user config option, default `.relay/handoffs`
- **Fix:** `/handoff` wrote to `.claude/handoffs/`, which Claude Code treats as protected — the
  write prompted for permission on every run and failed outright in non-interactive sessions

## 1.0.0

First public release. Three relay rituals (`/afk`, `/handoff`, `/yourturn`) plus two supporting
skills (`/project-sync`, `/journal`), extracted from a working two-machine, multi-tool setup and
generalised behind a per-repo config file.

- Per-repo `relay.config.md` replaces hardcoded project, service and path tables
- Machine-level values (vault root, fallback repo root) move to plugin user config
- Environment-specific checks split out into six opt-in adapters, all off by default
- Skills carry English instructions; README ships in English and Traditional Chinese
