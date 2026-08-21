# Changelog

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
