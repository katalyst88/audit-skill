# Proving the audit can fail

A clean pass is a statement about the checks, not about the thing. This file is how to earn the word
"clean".

## The method

1. **Pick one check per family in scope.** The one whose failure would matter most.
2. **Break what it watches** — not the check. In a copy, a scratch clone, a throwaway profile, or in
   place with an immediate revert you have already written down.
3. **Re-run the check through the whole path the real thing travels.** A fault injected into a
   generator's source proves nothing about an audit that runs on the shipped file; rebuild, then
   re-audit.
4. **Require it to fail.** If it still passes, the check is decorative — delete it or rewrite it, and
   say so in the report.
5. **Revert, and verify the revert** by reading the value back.

## What to break, by family

| Family | Inject |
|---|---|
| F1 unit | Flip a comparison operator, or widen a threshold by one |
| F2 integration | Point the base URL at a dead port; swap a key for a wrong one |
| F3 schema | Add the duplicate row the query is supposed to collapse |
| F4 edge state | Hand it zero rows, then 10,000, then a value exactly on the boundary |
| F5 acceptance | Change the behaviour the stated requirement names |
| F6 launcher | Disable the job, or stop the container, and see whether anything notices |
| F7 persistence | Fire the reverter you identified, then re-read |
| F8 backup | Age or truncate the newest archive and see whether the check still says healthy |
| F9 rendered artefact | Put a known word in, and a known-broken element, and confirm both are reported |
| F15 copy | Insert one "X, not Y" sentence and one uncontracted phrase |
| F16 accessibility | Set a foreground colour one step inside the contrast bar |
| F18 security | Request a path that must be refused, with no credential and with a wrong one |
| F20 evals | Add a case that must be refused, and one with no verdict at all |

## The coverage number

Any check that reads text out of something must print **how much it read**: `extracted chars / raw
chars`, rows seen / rows present, pages parsed / pages in the file. A checker pointed at a document
generator rather than the built document was seeing 52% of the text, and reported three companies as
absent from a document that names all three. The number would have shown that instantly.

Probe with a term you **know** is present. If a known term reads as absent, the extractor is broken,
not the subject.

## What the injection actually finds

Almost never an under-tested area. It finds checks that were **fake** — that could not have failed:

- an assertion on a value computed by the same code path it is checking
- a check that passes when its input is missing entirely
- a comparison of two siblings derived from one wrong source, which agree to 0.0 while both are wrong
- an existence check standing in for a visibility check
- a `try`/`except` that swallows the failure and carries on
- a status code standing in for a body

A suspiciously perfect result — `0.0`, `100%`, exactly equal, 31 of 31 — is a prompt to ask what both
sides were computed from. It is not a result. One layout fix asserted two elements agreed to a tenth
of a pixel, and both sat 28px from where the text actually started, because both were measured from
the same wrong source. The only thing that settled it was the leftmost ink on the rendered page.

## Worked precedent

A 26-check audit of a financial model passed with zero failures on the first run. A second script then
broke the file six ways — a section total repointed at the wrong rows, a tab hardcoded so it stopped
reading its source, stale commentary reintroduced, an input cell locked, a sheet left unprotected, a
tab severed from its input. All six were caught, and **that** is what made the clean pass believable.

On a later project, 31 of 31 injected faults were caught and the suite was green. Then the email it
produced was screenshotted: the top-ranked item was wrong, and every evidence quote on every card was
a raw character slice reading mid-word. Neither was a coverage gap. The tests asserted the rules
behaved as specified, and they did — **the specification was wrong**.

Mutation proves the checks match the spec. Only rendering the artefact tests the spec itself. Do both.
