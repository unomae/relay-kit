# Changelog

## 1.0.0

First public release. Three relay rituals (`/afk`, `/handoff`, `/yourturn`) plus two supporting
skills (`/project-sync`, `/journal`), extracted from a working two-machine, multi-tool setup and
generalised behind a per-repo config file.

- Per-repo `relay.config.md` replaces hardcoded project, service and path tables
- Machine-level values (vault root, fallback repo root) move to plugin user config
- Environment-specific checks split out into six opt-in adapters, all off by default
- Skills carry English instructions; README ships in English and Traditional Chinese
