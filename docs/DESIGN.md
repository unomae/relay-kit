# Design notes

Why the kit is shaped the way it is. Read this before forking or adding a check.

## 1. Read-only is a feature, not a limitation

Every ritual reports and stops. It would be easy to make `/afk` push for you, or `/yourturn`
stash your dirty tree before pulling. Both were deliberately not built.

The reason is asymmetry of cost. A missed warning costs you one confused minute at the other
machine. An automatic `git stash` on a tree you forgot about, or a push of a commit you were not
finished with, costs you an evening — and it happens on the run where you were *least* paying
attention, because you were leaving.

So: the rituals earn trust by being safe to run without reading the output first. Where a fix is
obvious they offer it and wait. The only unconditional write in the kit is `/handoff` creating a
new file, which cannot destroy anything.

**If you add a check, it reports. It does not fix.**

## 2. Pointer vs payload

The rule that keeps this kit from turning into a second project tracker:

> Anything already on disk gets referenced by path. Only what exists nowhere else gets written out.

A handoff note that copies the contents of your plan document creates a second source of truth,
and the copy starts rotting immediately. Worse, the next session cannot tell which one is current.

This is also why `/afk`'s drift check only ever warns. The moment a ritual starts *editing* your
plan document to keep it in sync, the plan document stops being something a human wrote and
becomes something generated — and nobody trusts generated state documents.

## 3. Config is Markdown, and nothing parses it

`relay.config.md` is a Markdown file with tables, read by the agent the same way a human reads it.
There is no schema and no parser. That is intentional:

- A malformed row degrades into "I could not read that" rather than an exception, so a typo cannot
  take down the whole ritual.
- The same file is useful to a person onboarding onto the repo. A JSON config would need a
  human-readable document beside it, and then those two would drift.
- New columns can be added without a migration. The agent reads what is there.

The cost is that you cannot validate it in CI. That trade is worth it for a file that exists to
orient a reader, not to configure a runtime.

## 4. Adapters default to off

Every check that cannot possibly fire in someone's setup is a line they learn to skip, and skipping
is a habit that generalises to the lines that matter.

So the base rituals contain only what is true for everyone with a git repo. Anything conditional —
nested checkouts, Windows symlinks, scheduled runners, a notes vault outside the repo — is an
adapter, off until turned on.

The same logic applies inside a check: **when there is nothing to report, report nothing.** A
section that only appears when it has content is a section people actually read.

## 5. A failed check must never look like a passed check

The most dangerous output this kit can produce is not an error. It is a clean report from a check
that silently never ran — a missing command swallowed by a redirect, a network call that timed out,
an empty variable that made a loop iterate zero times.

Every check therefore has three outcomes, not two: **found something**, **found nothing**, and
**could not tell**. The third is printed, never folded into the second.

This is also why `/yourturn` warns when a pull updated the relay setup itself: instructions load
before the pull, so that run used the previous version, and its output looks perfectly normal.
Normal-looking wrong output is the failure mode worth spending a whole check on.

## 6. The rituals do not call each other

`/afk` does not run `/handoff`. `/handoff` does not commit so that `/afk` has nothing to find.
They stay separate because they answer to different people at different moments: `/afk` serves the
person leaving *this machine*, `/handoff` serves the *next reader*, and `/yourturn` serves the
person arriving.

Chaining them would mean one ritual's failure silently degrading another's output, and it would
make each one harder to run alone — which is how they are actually used most of the time.
