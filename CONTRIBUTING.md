# Contributing

The most useful contribution is a new **adapter**: an optional check for a failure mode your
setup has and most setups do not. This document is mostly about how to write one.

Before anything else, read [`docs/DESIGN.md`](docs/DESIGN.md). It explains the invariants, and a
change that breaks one of them will be rejected however well it works. The short version:

> Rituals report. They do not fix. A check that fails must never look like a check that passed.

---

## Repo layout

```
.claude-plugin/marketplace.json     marketplace entry
plugins/relay-kit/
  .claude-plugin/plugin.json        plugin manifest + machine-level userConfig
  skills/<ritual>/SKILL.md          the five rituals
  adapters/<name>.md                optional checks
templates/relay.config.md           the per-repo config users copy
docs/DESIGN.md                      why the kit is shaped this way
```

There is no build step and no code. Everything an agent runs, it reads as Markdown.

---

## Adding an adapter

### 1. Check it belongs in an adapter

Adapters are off by default so that installing the kit does not hand someone a screenful of checks
that can never fire in their setup. The test is simple: **if most users would see this check do
something useful, it belongs in the ritual itself, not in an adapter.**

If the check only makes sense when you have a nested checkout, a Windows-flattened symlink, a
notes vault outside the repo, or some other thing you have and your neighbour does not, it is an
adapter.

### 2. Write `plugins/relay-kit/adapters/<name>.md`

Follow the shape of the existing six. The headings are a contract, not decoration:

```markdown
# Adapter: <name>

**Used by:** `/afk`, `/yourturn`          ← which rituals may run it
**Turn it on if:** <the condition in the reader's own terms>

## Why

The failure this came from. Concrete, past tense, one paragraph. If you cannot name a failure
that actually happened, the adapter is probably not worth shipping.

## Detect

The exact commands, in fenced bash blocks. Assume nothing about the platform.

## Report

What to say when it finds something, and — just as important — what to say when it finds
nothing. "Say nothing at all" is a valid and common answer.

## Rules

The safety boundary for this specific check.
```

Add a `## Configure` section instead of `## Detect` if the adapter is driven by config rather
than by a shell probe (see `vault-journal.md`).

### 3. Register it in three places

| File | What to add |
| :-- | :-- |
| `templates/relay.config.md` | A row in the **Adapters** table, `Enabled` set to `no` |
| `README.md` | A row in the Adapters table |
| `README.zh-TW.md` | The same row, in Traditional Chinese |

If your adapter needs a machine-level value (a path outside the repo, a token, a mirror
location), add it to `userConfig` in `plugins/relay-kit/.claude-plugin/plugin.json` rather than
to the repo config. Per-machine values do not belong in a file a team shares.

### 4. Obey the same rules as the rituals

- **Read-only.** Never commit, push, stash, reset, or delete. Not even inside a nested repo, not
  even when the main repo is being committed.
- **Silent when clean.** An adapter that prints "nothing to report" on every run trains people to
  stop reading the output.
- **Unverified, never clean.** If the check cannot run — a missing command, a repo that will not
  answer — say so and continue the sweep. A swallowed error that prints as "all clear" is the
  exact failure this kit exists to prevent.
- **No cross-calling.** Adapters are called by rituals. They do not call rituals, and they do not
  call each other.

---

## Testing your change

Plugin skills load when a session starts. **Installing and then invoking in the same session will
not work**, and that looks exactly like a broken skill. Open a fresh session first.

Install from a local checkout:

```bash
claude plugin marketplace add <path-to-your-checkout>
claude plugin install relay-kit@relay-kit
```

`marketplace add --scope user` installs and enables in one step. `--scope local` only records the
marketplace, so you have to run `install --scope local` afterwards as well.

Then exercise the ritual headlessly, in a throwaway repo rather than one you care about:

```bash
claude -p "/relay-kit:afk" --allowedTools "Bash(git *)" Bash Read Grep Glob
```

`--allowedTools Bash` on its own does **not** get past the permission gate for `git fetch`; the
`Bash(git *)` entry has to be listed explicitly. Add `Write Edit` when testing `/journal` or
`/handoff`, which are the only rituals that write anything.

Check both JSON manifests still parse:

```bash
uv run python -c "import json,io;[json.load(io.open(p,encoding='utf-8')) for p in ['.claude-plugin/marketplace.json','plugins/relay-kit/.claude-plugin/plugin.json']]"
```

---

## Changing a ritual

Bigger ask, same principles. Two things to keep in mind:

- **Ordering is a design decision.** "Committed but not pushed" is ranked above everything else in
  `/afk` because that is the failure that actually strands people. If you add a check, say where
  it goes in the order and why.
- **The config has no parser.** Do not add one. A malformed row must degrade into "not
  configured", never into an exception.

Bump `version` in both `plugin.json` and `marketplace.json`, and add a CHANGELOG entry that says
what broke and what fixed it, not just what changed.

---

## Out of scope

- Anything that fixes, syncs, or cleans up on its own. This is deliberate; see DESIGN.md §1.
- A project-management layer. The kit points at your plan documents and does not replace them.
- A parser, schema, or lint for `relay.config.md`. See DESIGN.md §3.

---

MIT licensed. By contributing you agree your work ships under the same licence.
