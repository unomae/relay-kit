# Relay Config

> Read by the relay-kit rituals (`/afk`, `/handoff`, `/yourturn`).
> **Agents read this file the way a human does — never write a parser for it.**
> Every section is optional. Delete what you do not use; the rituals degrade to
> "not configured" and keep going rather than failing.
>
> Copy this file to `.claude/relay.config.md` in your repo and edit the rows.

---

## Projects

One row per project or package a handoff might land in. This is the single
source of truth for "where does someone new start reading?".

`First read` is the one document that describes current state. `Verify command`
is what proves the project still works, and `Success evidence` is what that
command must print — without it, "I ran the tests" is unfalsifiable.

| Project | First read | Verify command | Success evidence | Warnings |
| :-- | :-- | :-- | :-- | :-- |
| `api` | `api/README.md` | `pytest -q` | exit 0, `0 failed`, no skips | migrations hit the shared staging DB |
| `web` | `web/PLAN.md` | `npm test` | exit 0, all suites pass | e2e opens a real browser |

<!-- Keep rows short. If a project needs more than the five columns above,
     put the detail in a "Before you touch this" section at the top of its
     First read document, and leave the pointer here. -->

---

## Services

Long-running local processes. `/afk` warns you they are still up before you
walk away; `/yourturn` lists them so the next machine knows what to start.

| Port | Project | Start command |
| :-- | :-- | :-- |
| 5173 | `web` | `npm run dev` |
| 8000 | `api` | `uvicorn app.main:app --reload` |

---

## Adapters

Optional checks. Off by default — turn on only what your setup actually has, so
nobody installs this kit and gets a screenful of checks that can never fire.
Each adapter is documented in the plugin's `adapters/<name>.md`.

| Adapter | Enabled | What it adds |
| :-- | :-- | :-- |
| `nested-repos` | yes | Scans repos nested inside this one (vendored/submodule-ish checkouts the outer `git status` ignores). |
| `symlink-repair` | yes | Detects symlinks flipped into plain files by a Windows checkout, and offers to restore them. |
| `hooks-path` | no | Ensures `core.hooksPath` points at a versioned hooks directory so shared pre-commit hooks actually run. |
| `mirror-skills` | no | Syncs a skills directory into a second agent tool's mirror after pulling. |
| `queue-check` | no | Reports the state of a scheduled-automation queue file and its open PRs. |
| `vault-journal` | no | Journals live outside the repo (Obsidian vault, cloud-synced folder). Requires `vault_root` in plugin config. |

---

## Notes

Free text the rituals should surface but that has no table of its own —
cloud sync that needs a manual check before leaving, a machine-specific gotcha,
a service someone else owns. Keep it to a few lines.

- Confirm cloud sync has finished before leaving this machine.
