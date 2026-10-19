---
name: audit
description: Use when asked to audit, review, check, sanity check, stress test, break, adversarially probe, or sign off something, or asked "is this ready", "is this safe to ship", "what have I missed", "would this work for someone else" — about a script, scheduled job, container, service, public endpoint, database, document, deck, spreadsheet, email, or an AI feature.
---

# Audit

An audit is organised by **the kind of harm**, not by feature, and every finding carries the thing
that proves it. Two phases, in order: **select** what is relevant, then **prove** what you claim.

The 20 families live in `families.md`. Read it once you know which band applies — do not try to run
an audit from this page alone.

## Phase 1 — Select, and say what you ruled out

Answer these about the target from what you can observe, not from what it is called. Each `yes`
pulls in families. Nothing else runs.

| If the target... | families |
|---|---|
| is supposed to run on its own, or right now | **F6 launcher and liveness, first, before anything else** |
| holds configuration, or you are confirming a past fix | F7 persistence |
| holds state nobody can regenerate | F8 backup |
| holds logic with a threshold, parser, score or key | F1 unit, F4 edge state |
| crosses a boundary: API, DB, network share, queue, container | F2 integration, F3 schema |
| has a stated requirement anywhere (spec, docstring, the ask) | F5 acceptance |
| produces anything a human reads | **F9 rendered artefact**, F15 copy |
| has a UI | F10 journeys, F14 heuristics, F16 accessibility, F17 visual |
| is used by anyone other than its author | F11 cold start, F13 hostile persona |
| can have access withdrawn: token, quota, cert, licence, card | F12 restricted states |
| is reachable from outside, or holds a credential | F18 security |
| competes for the machine, the uplink, or a third party's patience | F19 performance |
| calls a model, or acts on a model's verdict | F20 evals |

**F4 applies to everything.** So does F9 the moment any output reaches a human eye — including an
email, a chat message, or a log a person reads.

Then write the selection down before you start: the families in scope, and **the families you ruled
out with the predicate that ruled each one out**. An audit that never says what it skipped cannot be
told from an audit that forgot.

## Phase 2 — Rails. Read before touching anything live

Assume the target is production. Both baseline audits used to build this skill broke their own
read-only constraint while being careful, and self-reported it afterwards.

- **A dry run is not side-effect free.** It appends to the log and fires its events. Before running
  anything, ask what it RECORDS: a seen list, a streak, a rate limit, a quota, an audit log, an
  entitlement, a sync token, a notification. Say up front which of those you will touch.
- **A read-only probe can still consume state.** A sync endpoint can hand out a pending item's
  entitlement; a health probe can write the very line a staleness check reads, so the checker then
  measures itself.
- **Never burst wrong credentials at a live gate.** One refused request proves the gate; fifteen
  proves nothing more and trains a rate limiter or a WAF on the owner.
- **Never audit from the admin account**, and never conclude from one. Where there is only one
  account and no unprivileged twin, say so and audit as that account rather than creating a second.
- **Do not probe an internal-looking path you found by enumeration.** That is how a client IP gets
  blocked, and probing the block can widen it.
- **No restarts, recreates, prunes or deletes** as part of an audit. A prune run while diagnosing is
  how stopped containers lose their definitions.
- **Read a credential into a variable; never type it into a command.** An echoed command line puts
  the secret in the transcript and in any log that captures it.
- Prefer a throwaway identity or a copy of the state database. If you did pollute something, back up,
  clean up conservatively and asymmetrically, and **say plainly in the report that you caused it**.

## Phase 3 — Run triage, offer deep

Run every **T** family in scope now. For each **D** family in scope, name it and what it would cost,
and run it on a yes. Do not silently drop a D family — an unoffered family reads as a clean one.

If you are a subagent and cannot ask, put the D families at the end of the report as a numbered offer
with its cost, rather than waiting or deciding for them.

Fan out to parallel subagents when families are independent (security, accessibility, persona and
performance usually are). Give each the rails above verbatim and ask for findings only.

## Phase 4 — Prove the audit can fail

**A clean pass is evidence the checks are weak until you have broken something and watched a check
catch it.** Before the word "clean", "healthy" or "no issues" appears anywhere in the report:

Pick one check per family in scope. Break the thing it watches and confirm that check fails. Report
which checks you proved and which you did not.

**When mutating production is forbidden — which is the normal case — clone instead of improvising.**
Copy the config to a scratch directory and run a throwaway on a spare port, or point the check at a
doctored copy of the input. Say in the report that the mutation ran on a clone, and name the checks
you therefore could not prove.

What this finds is almost never an under-tested area. It is a **decorative** check: one that was
always going to pass. See `proving.md`.

## The report

The report is these parts, in this order, and nothing else:

1. **Verdict** — one line. Ship it / fix these first / do not ship.
2. **Scope** — families run, families ruled out with the predicate, and what you touched that records.
3. **Findings**, worst harm first. Each one: the severity, the claim in one sentence, the failure it
   causes with concrete inputs, and the file and line or the request.
4. **Proof** — per finding, one of `VERIFIED` (what you ran or read, quoted) or `INFERRED` (and what
   would settle it). These must never share a bullet.
5. **Checks I proved can fail** — the list from phase 4, and what stayed unproven.

Severity is fixed, by harm, never invented per run:
`DATA LOSS` > `WRONG OUTPUT SERVED` > `SILENT FAILURE` > `EXPOSURE` > `DEGRADED` > `COSMETIC`.

Apply a pragmatism filter: separate a conversion-killing barrier or a real defect from a subjective
visual preference, and say which is which. Do not pad the list to look thorough.

## Red flags — stop

- About to write "clean", "healthy" or "all good" with nothing mutated
- Reporting on the source when a built artefact exists
- Never got behind the gate, or never rendered the page
- Checked the payload and not the launcher
- Two derived numbers agreeing exactly, treated as proof
- Running it "just once, read only" to see what happens
- A finding whose evidence is your own knowledge of how the tool behaves, filed as verified

## Rationalizations

| Excuse | Reality |
|---|---|
| "It's a dry run, it changes nothing" | It writes a log line and an event. Those are production. |
| "The tests all pass" | Tests inherit the code's blind spots. The spec itself can be the bug. |
| "Status code was 200" | A wrong path, a rejected field and a dead session can all answer 200. |
| "The config is correct on disk" | Audit what ships. A crossed port serves the wrong app at 200. |
| "No findings, so it's solid" | A clean first pass means the checks are weak. Mutate, then say that. |
| "I don't have the credential" | Find out where this project keeps them, and ask if you may use one. |
| "Rendering it is a formality" | It is where the bug that survives testing gets found. Every time. |
| "One more probe won't matter" | Ask what it records first. Then decide. |
