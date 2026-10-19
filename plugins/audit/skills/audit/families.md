# The 20 audit families

Each family gives **applies when** as an observable predicate, **run** as the method, and **proof** as
what makes a finding in that family believable. Tier **T** = triage (minutes, just run it).
Tier **D** = deep (name it, offer it, run on a yes).

The precedents in each family are real failures from production systems, with the subjects removed.
They are here because an agent under time pressure negotiates with a rule and does not negotiate with
a case where that rule was already paid for.

---

## Band 1 — Is it correct

### F1 Unit
Individual functions, classes and branches in isolation.
- **Applies when:** the target holds pure or near-pure logic worth pinning — a parser, a threshold, a
  classifier, a dedupe key, a date or money calculation.
- **Run:** find the decision points, then run whatever tests cover them. No test file for the subtlest
  logic in the target IS the finding.
- **Proof:** a test that fails when you break the line it covers. A green suite over code you never
  mutated proves the suite runs, not that it discriminates.
- **Tier:** T

### F2 Integration and API
Modules, services and databases working together; endpoint status codes and payloads.
- **Applies when:** the target crosses a boundary — an HTTP API, a database, a network share, a queue,
  a third-party service, a container-to-host hop.
- **Run:** exercise the real endpoint read-only. Prefer the literal `127.0.0.1` over `localhost`:
  `localhost` resolves to `::1` first on most systems, and a service whose bypass allowlist is IPv4
  only answers **403 to `localhost` and 200 to `127.0.0.1` from the same machine, same endpoint**.
  Where a container publishes a port, probe from the host AND from inside the container runtime; a
  disagreement between them is the bug.
- **Proof:** the response **body**, not the status code. A CMS can answer a wrong path with 200 and an
  HTML error page. A server can answer a token-less POST with 400, which reads like a rejected
  password. An API can reject a missing required field with 200 and the error in the page body. One
  returns a **non-JSON body on a successful PATCH**, so parsing it as JSON reports a completed write
  as failed.
- **Tier:** T

### F3 Architecture and schema
Data models: normalisation, missing keys, indexes, where state lives, scaling bottlenecks.
- **Applies when:** the target owns a database, a state file, or a schema anything else reads.
- **Run:** read the schema. For SQLite: journal mode, `integrity_check`, `foreign_key_check`, whether
  a reader opens `mode=ro` **without** `immutable=1` — that flag is the obvious one to add on a
  read-only mount and it makes SQLite ignore the WAL, serving stale data with no error. Check that
  every query meant to take one row per group actually groups.
- **Proof:** the query that returns the duplicate, the orphan, or the arbitrary row. State counts.
- **Tier:** T

### F4 Edge state and data volume
Empty, loading, error and extreme states: zero rows, one row, 10,000 rows, a 50-character name, a
24,000-word document, a 60-hour media file, a page boundary landing inside a group of ties.
- **Applies when:** always. No exceptions. This is where the expensive bugs live.
- **Run:** ask what the smallest and largest real value is, then exceed both. Look for the boundary
  that is **equal** rather than greater: a strictly-greater-than cursor paging on a timestamp, where
  the page boundary fell inside a group of identical timestamps, dropped the rest of that group
  permanently — four records stayed invisible to a syncing client for two months.
- **Proof:** the output at each extreme, looked at. A 24,000-word article, correctly classified as
  suitable for a young child, was served to one as a 186-minute read past 249 passing tests. The
  classification was right and the length was never considered.
- **Tier:** T

### F5 Acceptance and BDD
Whether it does the thing that was actually asked for, in GIVEN / WHEN / THEN terms.
- **Applies when:** there is a stated requirement anywhere — a spec, a runbook, a docstring, a README
  paragraph, or the sentence the person used when they asked for it.
- **Run:** write the criteria out from the stated requirement, then test each. Where the code and the
  doc disagree, that is a finding regardless of which one is wrong.
- **Proof:** the requirement quoted, beside the observed behaviour. Tests inherit the code's blind
  spots because they were written from the same understanding, so **the specification itself can be
  the bug** and no amount of testing against it will say so.
- **Tier:** T

---

## Band 2 — Does it actually run

### F6 Launcher and liveness
The trigger, not the payload: the scheduled job, boot hook, service, daemon, container, watchdog.
- **Applies when:** the target is supposed to run on its own, or is supposed to be running right now.
- **Run:** liveness FIRST — `docker ps`, `kill -0 $PID`, `pidof`, the scheduler's own state, the
  service manager. Then the whole chain from trigger to thing started: the job definition, the wrapper
  script, the interpreter path, the account it runs as. A **disabled** job writes no log at all, which
  reads identically to "nothing to report". A job running as the system account has no user PATH, no
  mapped drives and none of the per-user tooling a human shell has, so checks that pass by hand fail
  there for a structural reason. Learn which of your scheduler's non-zero result codes are benign
  before treating one as a failure.
- **Proof:** a heartbeat that is **separate from the outcome**. A frozen log and a frozen state file
  are a corpse and an idle loop at the same time — every observable symptom is identical, so reading
  the log harder will not tell them apart. Ask whether the process is alive before reasoning about
  its logic.
- **Tier:** T, and run it before anything else. A dead launcher makes every other green check
  meaningless. One device install reported 254 host tests and 55 device checks passing, every config
  file byte-identical to source, on a device where the app could not start at all: every check
  validated the payload and nothing checked the launcher. The green count then actively hid it.

### F7 Persistence and reverters
Whether a setting, patch or fix survives the things that undo it.
- **Applies when:** the audit touches configuration, or is confirming a previous fix.
- **Run:** name the plausible reverters and test each — a service restart, a container recreate, an
  automated image updater's nightly pull, an app-to-app config sync, a scheduler rewrite, a deploy,
  and **your own probe calls**. A test endpoint handed a payload containing an `id` may write.
- **Proof:** the value read back after each reverter, and again at the end of the session. A setting
  read back correctly once is not a fix: one was confirmed working, reverted within the hour by the
  session's own experimental calls, and the owner saw the alarm rather than the fix.
- **Tier:** T

### F8 Backup and rollback
Whether the thing can be restored, and whether the backup of it is real.
- **Applies when:** the target holds state nobody can regenerate, or the audit precedes a change.
- **Run:** look at the far end — the archive's age and size at the destination, the pre-change copy
  beside the file, the exported volume. Never the backup script's own word for it: a script killed by
  its scheduler's time limit raises no exception, so nothing is logged, alerted or cleaned up, and the
  failure is invisible. Copying a live database file by file while writes continue can produce a set
  that passes an integrity check while being logically wrong.
- **Proof:** a timestamp and a byte count from the destination. Better, a restore.
- **Tier:** T

---

## Band 3 — The thing people meet

### F9 Rendered artefact
The output as the human receives it: the page, the email, the deck, the PDF, the card, the reply.
- **Applies when:** anything the target produces reaches a human eye.
- **Run:** produce the real artefact and look at it. Screenshot with a headless browser driver, using
  a fresh profile each run. Rasterise the deck. For a word-processor file walk the paragraphs **plus
  every table cell**. Render the email. **Fetch the page through its gate with real credentials** —
  an audit that never gets behind the login has not seen the product.
- **Proof:** the image or the extracted text, read. When extracting, print the coverage number
  (`extracted chars / raw chars`): a checker pointed at a document *generator* instead of the built
  file was seeing 52% of the text, and reported three companies as absent from a document that names
  all three. A checker that cannot say how much it read cannot be trusted when it says clean. Probe
  with a term you know is present; if it reads as absent, the extractor is broken, not the subject.
- **Tier:** T, and the highest-yield family there is. It is what caught a static portal's only real
  bug — the title and the description running together on one line — while every automated check
  passed: zero horizontal overflow, every tap target over 48px, 200 through the gate. It is also what
  caught five store screenshots of the wrong screen, from a script that exited 0 and printed DONE.

### F10 E2E and UI journeys
Full user paths: sign-up, the CRUD loop, error states, navigation.
- **Applies when:** the target has more than one screen, or a form.
- **Run:** drive the whole journey with a browser automation tool, as a stranger would.
- **Proof:** the journey completed end to end with the artefact captured at each step. **Existence is
  not visibility** — an element satisfies `waitForExistence` while scrolled far below the fold, which
  is how a screenshot test passed and photographed the wrong part of the page. Scroll until the
  element is hittable, then capture.
- **Tier:** D

### F11 Cold start and the new user
Zero data, no account, nothing seeded.
- **Applies when:** the target has onboarding, an empty state, or any first-run path.
- **Run:** a fresh unprivileged identity, or a guest profile if one exists. **Never audit from the
  admin account** — it hides every onboarding bottleneck and every permission gap.
- **Proof:** what the first screen actually says when there is nothing to show.
- **Tier:** D

### F12 Restricted access and billing states
Expired, failed, cancelled, locked out, quota exhausted, trial over, offline.
- **Applies when:** the target has a subscription, a licence, a token with an expiry, a quota, or any
  state where access can be withdrawn.
- **Run:** simulate the withdrawn state. Does it degrade gracefully, or hold the user's data hostage
  behind a wall? The common forms are a revoked or deleted API key, a lapsed subscription, an expired
  certificate, an exhausted CI quota, a dead upstream session, and an app signed out of its own cloud
  with the network down.
- **Proof:** the behaviour in the restricted state, observed, plus what the user is actually told.
- **Tier:** D

### F13 Hostile persona
A tech-averse, impatient, distracted or sceptical user, played in character.
- **Applies when:** the target is used by anyone other than you.
- **Run:** adopt the persona and stay in it. Count the clicks to the goal. Log every word of jargon.
  Record the exact point you would have given up.
- **Proof:** a click count and the point of abandonment. **Flag anything over 5 clicks as red.**
- **Tier:** D

### F14 Heuristic and flow evaluation
Nielsen's heuristics, cognitive load, error prevention, feedback visibility, jobs-to-be-done path.
- **Applies when:** the target has an interface at all.
- **Run:** state the core job, walk it step by step, and flag every step that needs knowledge the
  screen does not give. For a deeper pass, write the moderated usability script: task scenarios,
  think-aloud prompts and success metrics for the actual demographic.
- **Proof:** the numbered step list with the break named.
- **Tier:** D

### F15 Copy and microcopy
Button labels, error messages, tooltips, subject lines, the words in a reply or an email.
- **Applies when:** the target writes anything a human reads.
- **Run:** read it aloud in the reader's voice. **Count the "X, not Y" construction** — it is the
  machine-written tell, and a vocabulary check passes clean while it runs eighteen times in two
  documents. Keep contractions: the uncontracted form is most of what makes a sentence read as
  generated. Match the person's own voice if it goes out in their name, and have them see it first.
- **Proof:** the quoted line and the rewrite.
- **Never:** quote a user's own behavioural data back at them — their watch history, their timings,
  their counts. It reads as surveillance rather than service, and on anything shared it can leak what
  somebody else did. Put the evidence in the log, where it belongs.
- **Tier:** T when the target sends mail or messages, D otherwise.

### F16 Accessibility
Contrast, keyboard path, screen-reader order, focus, forced-colors, tap targets.
- **Applies when:** the target renders HTML, or a document anyone else opens.
- **Run:** 4.5:1 for text, 3:1 for large text and non-text UI, in light mode, dark mode and
  forced-colors. Walk the keyboard path. Check the heading hierarchy and that tables carry real
  headers with scope.
- **Proof:** measured ratios, and the keyboard path actually walked.
- **Tier:** T for contrast (mechanical, cheap). D for the full sweep.

### F17 Visual regression
Layout and interactive state across widths and devices.
- **Applies when:** the look has changed, or the thing is used on a phone.
- **Run:** screenshot at 390, 768 and 1440 wide, and compare to the previous shot if one exists.
- **Proof:** the pair of images, looked at. Zero horizontal overflow and tap targets over 48px can
  both pass while two text elements render on one line as a single run-on sentence.
- **Tier:** D

---

## Band 4 — Harm and limits

### F18 Security and vulnerability
Hardcoded secrets, injection, broken auth, exposure, OWASP Top 10, token handling.
- **Applies when:** the target is reachable from outside, holds a credential, or writes to something
  that is.
- **Run:** check the gate from outside the gate. Is there a login at all; is the **asset** path gated
  as well as the page (a login on a page whose images and manifest sit on an open path is not a
  login); is the credential in a file that rides a backup or a repo, or injected as environment.
  **Prove an install-default password does not work** — with a CSRF token where the app needs one, and
  against a known-rotated instance as a control, or a 200 on the login page cannot be told from a
  pass. Check for a secret committed to a public repo, and remember that a scanner which learns what
  secrets look like shares its own blind spot with the check that verifies it: confirm by cloning the
  published repo back and grepping for live values.
- **Proof:** a request that was refused, and the same request with credentials that was served.
- **Never:** burst wrong passwords at a live gate, probe an internal-looking path found by link
  enumeration, or draw conclusions from an admin session. See the rails in SKILL.md.
- **Tier:** T

### F19 Performance, capacity and cost
Latency, throughput, CPU, memory, rate limits, quota, token and dollar spend.
- **Applies when:** the target competes for the machine, the uplink, or a third party's patience.
- **Run:** take the total from the OS performance counter and attribute it with a sampling profiler
  over a real interval. Summing per-process CPU from a process list **undercounts badly** — one such
  sum read 61% against a true 100%, and missed the single biggest consumer entirely — and a
  cumulative CPU column is lifetime, not current, so it ranks any long-lived process top whatever it
  is doing now. For a network complaint, measure connection time **from outside** first: if connect
  is slow and first-byte is fast, the fault is the network path, not the application. Look for the
  forgotten process: a headless browser fleet or an emulator left running has no window, no taskbar
  entry and nothing watching for it, and one such pair accounted for 17% of a machine for 71 hours.
- **Proof:** a number with its unit and the window it was measured over. **Capacity is not the speed
  of one serial download** — the same segments over the same link gave 4.81 Mbps serially and
  23.05 Mbps over six connections, so a single-stream measurement will call a healthy link broken.
- **Tier:** T

### F20 LLM and AI evals
Accuracy, latency and edge-case robustness of a prompt, model or classifier: typos, sarcasm, rambling
input, adversarial phrasing, refusal behaviour, fail-closed behaviour.
- **Applies when:** the target calls a model, or acts on a model's verdict.
- **Run:** build a labelled set with the hard cases in it, and a rubric. Include the inputs that
  should be **refused or withheld**, not only those that should succeed — a fail-closed design is only
  proven by the case it must refuse. Test the no-verdict case explicitly: an item the model never
  rated must come out unavailable, not default-available.
- **Proof:** a score against labels **not generated by the same model under test**, with the failures
  listed individually. Watch for a bare `except` around the model call: one hid a hard per-request API
  limit behind 4,000 anonymous errors, so the system looked merely slow.
- **Tier:** D
