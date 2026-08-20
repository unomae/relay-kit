# Adapter: vault-journal

**Used by:** `/afk`, `/yourturn`, `/journal`
**Turn it on if:** your daily notes live outside the repo — an Obsidian vault, a cloud-synced
folder, anywhere that is not under git.

## Why

Two stores with two different sync mechanisms. The repo syncs when you push; the vault syncs when
the cloud client feels like it. Leaving a machine with one of them half-synced is how you arrive
at the other machine and read yesterday's state as if it were today's.

## Configure

Set `vault_root` and `journal_dir` in the plugin's configuration (`/plugin` → Relay Kit →
configure). Entries resolve to `<vault_root>/<journal_dir>/YYYY-MM/YYYY-MM-DD.md`.

## In `/afk`

- Check whether today's entry exists; mention it once if not. Never insist.
- Remind the user to confirm the cloud client has finished syncing before they walk away. This is
  usually not detectable from the shell, so ask rather than guess.

## In `/yourturn`

- Find the most recent entry and read **only its frontmatter** — `tldr`, `todos`, `blockers`,
  `next_hint`. That is the whole point of the frontmatter, and it keeps the read cheap.
- No frontmatter (older entries) → fall back to scanning the body, and suggest adding frontmatter
  next time.
- No entry at all → not an error. Say so and continue on git evidence alone.

## Rules

- The journal supplements git; it never overrides it. If yesterday's entry says the work is on a
  branch and git says otherwise, **git is right** — the entry was written before the last thing
  that happened.
- Cloud-synced folders can hold placeholder files that are not actually on disk yet. A read that
  fails there is a sync problem, not a missing file: report it as "unverified" rather than
  "no journal".
- Never write into the vault from `/afk` or `/yourturn`. Writing is `/journal`'s job, with the
  user in the loop.
