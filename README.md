# audit-skill

A skill for Claude Code that turns "can you audit this?" into something with a method behind it.

Ask Claude to audit a script, a service, a page, a document or an AI feature and you usually get a
reasonable-looking list. The problem is what is missing from it. Left to itself, a model picks two or
three familiar categories, reports on the source rather than the thing that ships, never gets behind
the login, and finishes with "no major issues" having broken nothing to see whether its own checks
could have caught anything.

This skill fixes the selection and the proof.

## Install

```bash
claude plugin marketplace add katalyst88/audit-skill
claude plugin install audit@audit-skill
```

Claude then loads it on its own when you ask for an audit, a review, a stress test, or whether
something is ready to ship. You can also invoke it directly with `/audit:audit`.

Without Claude Code, copy `plugins/audit/skills/audit/` into your assistant's skills directory.

## What it does

**Phase 1 — select.** Twenty families are listed, grouped into four bands: is it correct, does it
actually run, the thing people meet, and harm and limits. Each family has an observable predicate, so
selection is mechanical rather than a matter of taste. A static page does not get a unit-test pass; a
cron job gets its launcher checked before anything else. The audit has to state the families it ruled
out and the predicate that ruled each one out, because an audit that never says what it skipped reads
exactly like one that forgot.

**Phase 2 — rails.** The target is assumed to be production. A dry run writes a log line and fires an
event. A read-only health probe can write the line a staleness check later reads, so the checker ends
up measuring itself. Fifteen wrong passwords at a live gate prove nothing the first one did not, and
train a rate limiter on the owner. These are explicit, because both baseline runs that produced this
skill broke their own read-only constraint while being careful about it.

**Phase 3 — triage, then offer.** Cheap high-yield families run immediately. Expensive ones get named
with their cost and run on a yes, so an audit is minutes rather than an afternoon.

**Phase 4 — prove the checks can fail.** Before the word "clean" is allowed in the report, one check
per family has to be broken and watched to fail, on a clone where production cannot be touched. What
this step finds is almost never an under-tested area. It finds checks that were decorative: an
assertion on a value computed by the code it is checking, an existence check standing in for a
visibility check, a status code standing in for a body.

The report shape is fixed, severity is fixed by harm, and every finding is labelled `VERIFIED` with
what was run, or `INFERRED` with what would settle it. Those two never share a bullet.

## The twenty families

| Band | Families |
|---|---|
| Is it correct | unit · integration and API · architecture and schema · edge state and data volume · acceptance and BDD |
| Does it actually run | launcher and liveness · persistence and reverters · backup and rollback |
| The thing people meet | rendered artefact · E2E journeys · cold start · restricted access and billing · hostile persona · heuristic and flow · copy and microcopy · accessibility · visual regression |
| Harm and limits | security and vulnerability · performance, capacity and cost · LLM and AI evals |

The first eight bands' worth of engineering families are the familiar ones. The four that earn their
place are **launcher and liveness**, **persistence and reverters**, **rendered artefact** and
**edge state** — the families that get skipped precisely because they look too boring to fail.

## How it was built

Test-driven, following the `superpowers:writing-skills` method.

Two fresh agents were given a real script and a real gated web service and asked to "run an audit",
with no skill present. Their behaviour was recorded, and that is the baseline this skill is written
against. Both broke their stated read-only constraint. Both audited files on disk rather than what was
served. Neither proved a single check could fail. One listed "rendering" among the things it skipped,
on the one target whose only real bug had historically been found by rendering it.

The same service was then audited again with the skill loaded. It selected eight families and ruled
out ten with the predicate for each, rendered both pages behind the gate at two viewport widths,
byte-compared what was served against what was on disk, sent one wrong-credential request rather than
sixteen, mutated two checks on a throwaway container to prove they discriminate, and found three real
defects the baseline had no route to: a double negative in the copy that stated the opposite of what
was meant, a 21px tap target, and a link rendering in browser-default blue against the palette.

Six wording problems it reported afterwards were fixed before release, including what to do when
mutating production is forbidden, and how to offer a deep family when you are a subagent and cannot
ask.

The war stories inside the skill are real production failures with the subjects removed. They are
there because an agent under time pressure will negotiate with a rule and will not negotiate with a
case where that rule was already paid for.

## Licence

MIT.
