---
name: project-sync
version: 1.0.0
description: |
  Bring the current git project up to date and scan its structure, so later work starts from
  reliable context. Triggers on "sync the project", "pull latest and scan the structure",
  "re-scan this project's architecture", "sync <path>".
  With no path given it uses the current working directory only — it never searches by project
  name and never falls back to a default project.
user-invocable: true
tags:
  - workflow
  - git
  - context
---

# Project Sync

Aligns a project to its latest state and reports the structure, so that whatever comes next —
analysis, edits, a handoff — is standing on current ground. `/yourturn` builds on this.

## Target resolution

1. An explicit path from the user wins.
2. Otherwise, use the current working directory if it is a git repo.
3. If it is not a git repo, stop and ask for the full path.

Never search the workspace by project name. Never apply a default project. Never change
directory on the user's behalf.

## Workflow

### 1. Preflight

```bash
git branch --show-current
git status --short --branch
git log -1 --oneline
```

Confirm the path exists, is a git repo, and note the branch, whether the tree is clean, and the
latest local commit.

### 2. Dirty tree policy

If the working tree is not clean: no reset, no checkout to discard, no automatic stash, and never
overwrite uncommitted work.

If the user only said "sync", report the dirty state and ask whether to pull anyway. If they
explicitly said "pull latest", prefer `git pull --ff-only`.

### 3. Pull

```bash
git pull --ff-only
```

Fall back to plain `git pull` only where `--ff-only` is unavailable. On conflict, divergence or
any other error: stop, report the error and the affected files, and resolve nothing automatically.

### 4. Post-pull check

```bash
git status --short --branch
git log -1 --oneline
```

Report the pull result, the latest commit, whether the tree is clean, and anything the user needs
to deal with.

### 5. Scan

Skip `.git`, `node_modules`, `.venv`, `venv`, `__pycache__`, `.pytest_cache`, `dist`, `build`,
`.next`, `coverage`, `target`, `vendor`.

Identify: backend, frontend, shared contracts and types, tests, docs, scripts, config, fixtures,
and generated artifacts.

Read as needed: `README.md`, `CLAUDE.md`, `AGENTS.md`, `package.json`, `pyproject.toml`,
`requirements.txt`, build and test configs, and any design or spec documents.

Do not read large binaries. Do not modify a single project file as part of syncing.

## Output

Answer in the user's language.

```markdown
**Sync result**
- Path: | Branch: | Latest commit:
- Pull: | Working tree:

**Structure**
- Backend: | Frontend: | Docs: | Tests: | Fixtures:

**Worth noting**
- ...

**Suggested next**
- ...
```

## Rules

- No destructive git commands, no automatic stash, no automatic branch switching.
- Never resolve a conflict automatically.
- Never overwrite uncommitted work.
- If the sync fails, stop and say so. Do not report a sync that did not happen.
