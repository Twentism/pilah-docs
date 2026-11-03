# IR Part C — soft skills

Organised into the same **week slots** as [Part B](part-b.md). The last two
sit exactly in their claim windows; the two before them were written before
those windows were fixed and report work a few days later than their heading
suggests. The dates in the table are the real dates of the real work.

Part C is thinner per
week, so the weeks are sections here rather than separate pages — there is not
enough in any one of them to justify its own page, and splitting would pad
rather than clarify.

| Week | Dates (2026) | What it shows |
|---|---|---|
| [Sprint 1 · Week 1](#sprint-1-week-1) | 15–21 Sep | Learning decimal rounding modes properly |
| [Sprint 1 · Week 2](#sprint-1-week-2) | 22–28 Sep | Asking instead of guessing; reviewing where I add value |
| [Sprint 1 · Week 3](#sprint-1-week-3) | 6–12 Oct | Worktrees and stacked branches; Flutter web; four blockers reported |
| [Sprint 2 · Week 1](#sprint-2-week-1) | 12–16 Oct | Waiting for permission, then fixing a blocker that was finally mine |
| [Sprint 2 · Week 2](#sprint-2-week-2) | 13–19 Oct | A defect my own refactor created, and a reviewer who found it |
| [Sprint 2 · Week 3](#sprint-2-week-3) | 20–26 Oct | Disagreeing with an approver, with evidence; seven findings answered |
| [Sprint 3 · Week 1](#sprint-3-week-1) | 27 Oct – 2 Nov | Two framework mechanisms behind one button; reviewing a teammate's code I had never touched |

---

## Sprint 1 Week 1

### Learning beyond the classroom — decimal rounding modes

PIL-168 looked like a one-line rounding change and was not. `ROUND_HALF_UP`,
`ROUND_DOWN` and `ROUND_FLOOR` differ in ways that matter for money:
`ROUND_DOWN` truncates toward zero, `ROUND_FLOOR` toward negative infinity, and
they diverge on negative values. Choosing one meant understanding what a
fraction of a rupiah *means* in a system migrating legacy balances that carry
sen, not just which function name to type.

It paid off later in a review: because I knew the mode mattered, I checked
Vegard's refactor by comparing the helper's actual quantize call against what
it replaced, rather than trusting the PR description.

### Working with the team

The rounding rule — whole rupiah, always **down** — came from Tristan in
conversation, not from the PRD, whose PIL-138 notes implied `ROUND_HALF_UP`.
Starting from the written spec alone would have shipped the wrong rule. An
argument for asking, not for reading more carefully.

---

## Sprint 1 Week 2

### Working with the team — asking instead of guessing

On PIL-223 the `email` field could not be locked yet, because the PR making it
the account-linking key had not merged. I raised it with Pascal, we agreed to
defer, and it landed in PIL-288 — rather than implementing a lock whose
rationale did not exist yet.

The scope narrowed mid-ticket for a reason that is invisible in the code. That
is exactly the kind of thing that has to come from a person.

### Reviewing where I can actually add value

Reviews on be #48 and #49, both on Heraldo's work. The pattern that held all
project: I review well when the PR touches code I wrote, and less often
otherwise. Good reviews, narrow coverage — stated again as a limitation in
[Part B · B4](b/s1-w2.md#b4-peer-review).

### Being wrong

!!! failure "I asserted a merge conflict that did not exist"
    I said two PRs conflicted. Testing it with `git merge-tree --write-tree`
    returned clean — they touched different regions of the same file. I had
    inferred the conflict from the file list instead of checking.

---

## Sprint 1 Week 3

Sprint planning set the scope this week had to respect: web supports **Super
Admin, Pengurus and Pengurus Induk only**, never Nasabah. One sentence in a
planning meeting that shaped every role check in PIL-293.

### Learning beyond the classroom — worktrees and stacked branches

PIL-293 depended on a fix that had not merged: staging's Flutter web build
compiled and then died with `FirebaseOptions cannot be null`. Waiting would
have cost days, so the branch was based on Tristan's unmerged branch instead.

That required things I had not used — worktrees so several branches can be
checked out at once, `git rebase --onto` to re-parent a stack when the base
moves, and `--force-with-lease` to push a rewritten branch without clobbering
anyone.

Exercised twice. Tristan force-pushed his base, then his work squash-merged
into staging — which changes the lineage while leaving the tree identical.
Confirming that before rebasing is what made it safe:

```bash
git merge-base --is-ancestor origin/feature/pil-334 origin/staging   # no
git diff --stat origin/feature/pil-334 origin/staging                # empty
git rebase --onto origin/staging origin/feature/pil-334 feature/pil-293
```

Different lineage, identical content — so a 21-commit rebase with zero
conflicts. **A squash merge makes `git log` disagree with `git diff`, and the
diff is the one that decides whether a rebase will hurt.**

The same knowledge paid off directly at the end of this week. Merging #70 with
a **merge commit rather than a squash** kept PIL-293's commits as ancestors of
`staging`, so stacking PIL-340 on top became a plain `git rebase` with no
surgery — and it kept 35 red/green/refactor commits visible in `git log
staging`, which is the B1 evidence.

### Learning beyond the classroom — Flutter web as a compile target

Not a separate app: the same Dart compiled to JavaScript, painted to a canvas.
Consequences learned the hard way — browser automation snapshots come back
empty because there is no DOM, so clicking has to be done by coordinate; and
`kReleaseMode` compiles the demo login out, which makes release-build QA
impossible without a real OAuth credential. This week that meant serving the
release build to prove it *renders*, then doing the authenticated pass on the
debug server.

Also learned this week, from a bug: **CORS is a browser-only mechanism.** It is
why a header that had worked in the native app for months blocked every web
save, and why nobody had seen it.

### Working with the team — reporting blockers instead of routing around them

Four this sprint: a missing `google-services.json` blocking the APK check, no
web OAuth client ID blocking release QA, a DI regeneration defect in somebody
else's merged code, and the CORS gap.

I fixed none of them properly, because none were mine to fix. Two are worth
separating:

- The **DI defect** is invisible to CI — every gate passes because CI never
  *runs* the app — so saying nothing would have left a trap for whoever ran
  `build_runner` next.
- The **CORS gap** I did patch, but locally and uncommitted, because the team
  confirmed it was already being handled alongside a web deployment. Fixing it
  properly in a PR would have duplicated someone's in-flight work. Tristan
  later cleared me to fix it for real, which became PIL-354 — see
  [Sprint 2 Week 1](#sprint-2-week-1).

### Handling review findings

@HeraldoArman raised three findings on #70 and approved anyway. I fixed all
three in the PR rather than taking the offered follow-up, answered each
in-thread, and resolved them.

I also **declined one part** — merging the rail and drawer header *styling* —
because they are not parallel, and unifying them means designing a new widget
rather than moving code. I said so in the thread. Agreeing to everything a
reviewer suggests is not collaboration either.

The most useful finding was the one that cost me something: his note that a
widget with no call sites is tested only against itself exposed a design claim
I had published days earlier and not re-checked.

### Being wrong

!!! failure "I reported a working app from a clean log"
    Twice I said the web app booted, based on build logs with no errors. The
    page was blank white. A clean log is not a rendered page. I now verify with
    a screenshot of the actual UI — which is why this week's release build was
    served and photographed rather than just compiled.

!!! failure "I claimed nine failures were pre-existing"
    Seven were. Two were my own regression, in files my verification had not
    touched — and I had said "verified". CI contradicted me and was right. A
    claim to have verified something has to name what was verified.

!!! failure "I called a current tool deprecated"
    Dismissed a CLI as abandoned because its version number was `0.1.22`. That
    was its own versioning scheme; the tool was current.

!!! failure "I missed the OWASP requirement twice, then under-claimed it"
    PIL-168 had it in the PR body instead of the commit. PIL-293 had it nowhere
    until caught after the PR was green. PIL-340 named three items where the
    code supports four — A08 is genuinely present and was not named at commit
    time. Three strikes for one remedy, each softer than the last, which is
    evidence the remedy does not hold rather than evidence it is improving.

What connects these: every one came from inferring a conclusion that was cheap
to check directly. Logs instead of the screen, a file list instead of a merge
test, a version number instead of the docs. The habit I am building is to check
the thing itself when checking is cheap — and when it is not cheap, to say
which one I did.

---

## Sprint 2 Week 1

### Learning beyond the classroom — what CORS actually negotiates

I had treated CORS as one switch: origins in or out. It is two separate
negotiations, and this week needed both.

**Request headers** are negotiated *before* the request exists. The browser
sends a preflight `OPTIONS` asking "may I send these headers?", and if the
answer omits one, the real request is never sent at all — which is why the
backend log showed `OPTIONS … 200` four times and no `POST`. The server never
refused anything; it was never asked.

**Response headers** are negotiated *after*. The response arrives complete, and
`Access-Control-Expose-Headers` decides which of its headers JavaScript is
allowed to read. A download can therefore succeed while the client cannot see
the filename it came with.

The practical lesson is the asymmetry: `CORS_ALLOW_ALL_ORIGINS=true` sounds
total and governs neither of these. Permissive on one axis is not permissive on
the others.

### Working with the team — waiting for permission, then taking it seriously

I found this bug in Sprint 1 Week 3 and deliberately did not fix it, because
the team said a web deployment was already touching that area. Patching someone
else's in-flight work is not helpfulness.

When Tristan cleared it, the scope grew rather than shrank: not the one line I
had patched locally, but the reason two files could disagree about a header name
in the first place. Having waited for the go-ahead, the right response was to
fix the cause rather than re-apply my workaround.

The same restraint applied twice more in the same change:

- `CORS_ALLOW_ALL_ORIGINS` still defaults to `true`, which is a wider exposure
  than the one I was fixing, on a file I had open. It belongs to PIL-334 and a
  deployment is in flight, so I **flagged it in the PR** instead of changing
  it. Noticing it silently would have been the real failure.
- The OWASP count stayed at three items when five would score a level higher,
  and the PR stayed one MR when three would score a level higher. Both were
  available by splitting or padding. Neither is work.

### Being wrong

!!! failure "I planned the issue under the wrong parent"
    I had the new issue down as a sibling of PIL-340, and a `blocks` relation
    pointing at PIL-335. Querying Linear before creating anything showed
    PIL-340's parent is **PIL-346**, and that PIL-335 was already **Done** — a
    blocking relation on a finished issue is noise. Both facts were one query
    away and I had carried them for days from an earlier conversation.

!!! failure "I accepted a secondhand claim about a tool"
    I was told a Sonar CLI skill had been added to `AGENTS.md`. I looked: not
    in either repository's current `staging`, not in `.agents/skills/`, not in
    the global skills, and no `sonar-scanner` or `SONAR_TOKEN` present. The
    useful answer was "it is not there, here is what I checked" rather than
    improvising a command that would have failed. What the lead dev probably
    meant is real and better: pushing a branch triggers SonarCloud without a PR.

!!! failure "I followed a documented step that was wrong for this repository"
    The `ship` skill copies `.agents/` into a new worktree because it is
    "intentionally gitignored". In `pilah-be` it is **tracked**, so my copy
    silently overwrote the current skill with a 125-commit-old version. Caught
    only because I read `git status` afterwards. A step being written down does
    not make it safe in a repository it was not written for.

These are the same failure as the earlier weeks' — a conclusion taken on trust
when checking was one command away. The difference this week is that I checked
*before* acting on two of them rather than after being contradicted.

---

## Sprint 2 Week 2

### Learning beyond the classroom — an async result can outlive the state that started it

A bottom sheet lives for exactly one row. A side panel does not: selecting
another row refills the same form. Moving a form from one container to the
other changes its **lifetime**, and I did not think of that as a change at all
— the extraction was behaviour-preserving by every test I had.

It is not. A save dispatched for row A can return after the panel has moved to
row B, close the panel the user is filling, and attach a server field error to
a field that does not belong to it. The remedy is a generation counter: record
the row's generation before the request, compare after, and suppress only the
parts that touch the form — the success notification still shows, because the
request really did happen.

The general rule I took from it: **a component reused across containers
inherits the longest lifetime of any of them**, and every `await` inside it
then has to ask whether the world it returns to is the one it left. That is not
a Flutter fact; it is the same shape as a stale HTTP response, or a cancelled
job writing its result anyway.

### Working with the team — a comment that arrived too late to act on in place

TrayMachi left a `[P2]` on PIL-339's PR after it had already merged. The cheap
response is to fold the fix into whatever branch is open next, where it
disappears into a feature diff and the reviewer never sees an answer.

It went out as [its own PR](b/s2-w2.md#a-panel-that-outlived-its-window)
instead — four commits, with tests that fail without the fix. A late review
comment is still a review comment, and the person who wrote it should be able
to see what happened to it.

### Being wrong

!!! failure "My refactor created the bug, and a reviewer found it, not my tests"
    The extraction in `0eab55fb` passes every test it had and is still the
    change that made a late save able to close another row's panel. Every test
    I wrote held the selection still, so the suite was structurally blind to
    it. Scallss found it by reading the PR. It is credited in the commit body
    of the fix, because the alternative — fixing it quietly — would make my own
    record look better than it was.

!!! failure "I read a green tick instead of the issue count"
    The first push of PR #126 was green on CI with **five** open SonarCloud
    issues. This is the exact lesson PR #70 had already taught me: the quality
    gate can pass while issues remain. I had written that down and still read
    the tick.

---

## Sprint 2 Week 3

### Learning beyond the classroom — what a CSP nonce does and does not authorise

Four things about Content-Security-Policy that I had wrong or had never
considered, each of which changed the implementation:

- **CSP fails open.** Browsers ignore directives they do not recognise, so
  `frame-ancestor` instead of `frame-ancestors` yields a header that looks
  right, passes any test matching on text, and protects nothing. That single
  fact is why the policy is assembled by a Builder with one method per
  directive rather than written as strings.
- **A nonce authorises a `<style>` *element*, never a `style=` *attribute*.**
  Attributes can only be permitted by `'unsafe-inline'`, which is exactly what
  the policy exists to avoid. Two inline style attributes had to move into a
  nonced block instead of being accommodated.
- **A signature proves integrity, not safety.** `TimestampSigner` proves the
  server produced a value; it says nothing about whether printing that value is
  safe. The signed `next` parameter did not block the injected payload — it
  laundered it.
- **Middleware ordering decides whether a response gets a header at all.** A
  layer that returns early never reaches the view, so anything below it never
  runs. Choosing the policy in the *response* phase is what makes the outermost
  position safe.

Also learned by using it: `drf-spectacular` ships
`SpectacularSwaggerSplitView` specifically for strict CSP — it serves the
initialisation script as its own file instead of inline. The upstream answer
existed and was better than the workaround I would have written.

### Handling review findings — disagreeing with an approver, with evidence

The most useful finding of the week came with a fix attached, and the fix does
not work. The suggestion was to set `response.csp_nonce` in the developer page
the way the OAuth popup does; doing that selects the closed-document policy
`default-src 'none'`, which blocks the page's CDN assets and its Google script.
It trades one dead page for another.

Two things mattered in how that was handled. First, it was **applied and
loaded** before being argued with — the finding was right and the remedy was
wrong, and only running it shows which is which. Second, the reply says where
it deviated from the suggestion and why, rather than silently implementing
something different and marking the thread resolved. The reviewer had already
approved; their finding still changed the code more than any other.

All seven findings across the two PRs were answered **in their own threads**,
each naming the commit that fixed it, and both PRs merged the same day. A
single summary comment would have been faster to write and much harder for a
reviewer to check.

### Working with the team — splitting so the urgent half can move

The XSS fix and the CSP policy touch the same function and could have been one
pull request. They were two, stacked, because one of them fixes a live
unauthenticated vulnerability on a deployed backend and the other is a policy
refactor that changes renderer configuration for the whole API.

That turned out to be the right read: the fix was approved with two docstring
nits, while the policy drew a medium finding that took a full revision to
answer. Bundled, the vulnerability fix would have waited for that.

It cost something too, and both costs are written down rather than left for the
next person to discover: SonarCloud cannot analyse a PR whose base branch is
not itself analysed, and the rebase at merge time changed every commit hash in
the second PR.

### Being wrong

!!! failure "I accepted a reviewer's fix before running it"
    It came from someone who had just approved the PR, it matched a pattern
    already in the codebase, and it was wrong for this page. The habit that
    caught it is the one this project keeps relearning: load the thing and look
    at it. Written up in
    [Sprint 2 · Week 3](b/s2-w3.md#the-review-round).

!!! failure "A contract my PR stated, and did not keep"
    The PR said "every response carries a policy". It did not: the middleware
    sat last, so the CORS preflight, the HTTPS redirect and the disallowed-Host
    400 all shipped without one. No test of mine could have caught it, and none
    did — alghani46 found it by reading `MIDDLEWARE`. Stating a contract is not
    the same as checking the places it has to hold.

!!! failure "I reported nine test failures that were four of my own making"
    My first full-suite figure for the week blamed the repository for failures
    my own `DJANGO_DEBUG=true` invocation had caused. Isolated afterwards by
    re-running on a clean worktree with the flag and then without it — which is
    what should have happened before reporting a number at all.

## Sprint 3 Week 1

### Learning beyond the classroom — one button, two different mechanisms

The side sheet had to close when the browser's Back button was pressed. I
assumed that was one problem. It was two, and neither is something I had met
before.

**First:** a panel opened with `showGeneralDialog` never enters go_router's
page list, so it creates **no browser history entry at all**. I measured it
rather than assuming — `history.length` stayed at 4 when the panel opened, and
the URL never changed — which meant Back was popping the *page underneath* the
panel. On a dirty form that navigated the whole shell away and destroyed what
had been typed.

**Second, and the part I would not have predicted:** making the panel a real
declared route fixed *where* Back lands, and the discard guard still did not
run. `PopScope` is a `PopEntry`, and only `Navigator.maybePop` consults it. A
browser history traversal does not pop anything — go_router's
`RouteInformationProvider` hands the new location to the delegate, the delegate
rebuilds the page list, and the page is removed **declaratively**. Nothing
calls `maybePop`, so the guard is never asked. The hook that does cover it is
`GoRoute.onExit`.

What I actually learned is smaller and more useful than either fact: a scrim
tap and a Back press look like the same action to a user and travel through
entirely different machinery. Fixing one says nothing about the other, and the
only way I found that out was by driving the built app and watching.

### Learning beyond the classroom — design patterns that are already there

I had been taught design patterns as things you build: write the abstract
creator, write the concrete subclass. Flutter had already written them.
`Page.createRoute` **is** Factory Method, and `PageRoute` filling in `opaque`,
`buildPage` and `buildTransitions` **is** Template Method, with the framework
owning the algorithm.

Recognising a pattern I did not write, and then deciding to participate in it
rather than hand-roll an overlay, turned out to be a different skill from
implementing one from a diagram. It also paid: `opaque => false` is a single
inherited hook, and it is the entire requirement of the ticket.

### Learning beyond the classroom — a measurement tool has a model of your repo

SonarCloud analyses **one** long-lived branch on this project's plan. My second
pull request is stacked on my first, so its real base is a branch Sonar has
never analysed — and rather than failing, it quietly fell back to `staging` and
reported a coverage number covering **both** pull requests' changes.

I found it by comparing the base Sonar recorded against the base GitHub shows.
The general lesson is the one worth keeping: a quality tool is not measuring
your repository, it is measuring its own model of your repository, and when
your branching does not fit that model it still returns a number. The number is
withheld from [B5](b/s3-w1.md#b5-code-quality) for that reason.

### Working with the team — reviewing code I had never touched

Earlier in this project I wrote down a limitation about myself: I review well
when a pull request touches code I wrote, and less reliably otherwise. Both of
this week's reviews were on **Vegard's** work, and the backend half was a
ledger change in an app I had never opened.

I reviewed it by tracing **every** `Transaksi` query site in `apps/` instead of
trusting the table in the description, and reported that no filter had been
missed — then raised three things it had not covered, including that
`Transaksi` declares no indexes at all while its sibling `Pencairan` does,
which bulk voiding turns from tolerable into a real cost.

Across the project that limitation has now closed properly: **12 of my
teammates' pull requests reviewed, covering all four of them** — Ghani four,
Pascal three, Vegard three, Heraldo two.

### Working with the team — saying what was not wrong

On the mobile half I went in looking for one specific defect class first:
async lifetime, the bug that had already bitten the harga panel, the jadwal
panel and my own side sheet in the same week. It came back **clean**, and I
said so in the review.

That is deliberate. A review that lists only problems leaves the author
knowing what I disliked and not what I actually examined — and "I checked the
thing that broke three other panels, and this one is fine" is more useful to
them than silence on it.

### Handling review findings

Four findings on the side-sheet pull request, from two reviewers, and the most
useful fact about them is that **two people found the same bug independently**:
with no guard attached, the panel could not be closed at all, by scrim, Escape
or Back. Tristan tagged it 🔴; Ghani reached it from a different direction. All
four were fixed test-first, each with its own failing commit before the fix.

On the Pengumuman removal, Vegard corrected a claim I had made in my own pull
request body — that the untouched test suites proved the feature was
restorable. They guard the *pages*; the route-level guards had gone out with
the builders. He was right, and the fix is now the best test on that branch.

### Being wrong

!!! failure "The most load-bearing fact in my plan was stale by the time I used it"
    The plan turned on seven cubits being provided above `MaterialApp.router`,
    which I had verified by reading the file. Overnight someone replaced Bloc
    with Riverpod and that provider ceased to exist. The conclusion survived,
    but by luck rather than method. On a repository where `staging` moves forty
    commits in two days, a fact verified on day one is not a fact on day two.

!!! failure "I wrote a test seam production was forbidden to use"
    I injected the platform through an `@visibleForTesting` parameter. That
    annotation means precisely *production must not pass this*, while `kIsWeb`
    being a compile-time constant means production is exactly what has to. It
    read as the tidy, disciplined choice and it was a back door. The project
    already had the right idiom — a plain explicit argument — three files away.

!!! failure "My own pull request body claimed something the browser disproved"
    It justified the container with "a real route: Esc, browser back and
    `Navigator.pop` all close it". Browser back did not close it. The claim was
    already sitting in a pull request awaiting review when I found out, by
    running the app rather than by re-reading the sentence.

!!! failure "I reported the Android build as blocked, and it never was"
    `JAVA_HOME` pointed at a JRE, which has no compiler, while a real JDK sat
    installed alongside it. I had recorded that build as impossible on this
    machine in an earlier week, and used it as a reason to skip verifying on a
    real device. This week it built in about six minutes.

---

## Across every week — AI literacy

I use Claude Code as a working tool, and the useful part is not the generation.
It is being a reviewer of output that is confidently wrong sometimes.

The full record, with my actual prompts at the points where I rejected or
redirected what I was given, is reported **inside each week** as that week's
B7 section — [Sprint 1 · Week 1](b/s1-w1.md#b7-ai-literacy),
[Week 2](b/s1-w2.md#b7-ai-literacy), [Week 3](b/s1-w3.md#b7-ai-literacy),
[Sprint 2 · Week 1](b/s2-w1.md#b7-ai-literacy),
[Sprint 2 · Week 2](b/s2-w2.md#b7-ai-literacy),
[Sprint 2 · Week 3](b/s2-w3.md#b7-ai-literacy) and
[Sprint 3 · Week 1](b/s3-w1.md#b7-ai-literacy). The tally by cause, which
only makes sense across all of them, is in
[Part B](part-b.md#b7-how-i-use-ai).

---

## Across every week — time allocation and output

The commitment criterion is scored on *"discipline allocating time per weeks"*
and on *"the quality and quantity of works"*, so both are measured here rather
than asserted.

### Hours

| Week (Mon–Sun) | Hours |
|---|---|
| 14 – 20 Sep | 5.2 |
| 21 – 27 Sep | 12.8 |
| 28 Sep – 4 Oct | 6.0 |
| 5 – 11 Oct | 15.3 |
| 12 – 18 Oct | 16.7 |
| 19 – 25 Oct | 3.2 |
| 26 Oct – 1 Nov | 11.2 |
| 2 – 3 Nov *(partial)* | 5.2 |
| **Total** | **75.5 hours** |

Spread over **35 distinct working days** inside a 49-day calendar span, 16
September to 3 November, across 16 working sessions.

!!! info "What this number is, and what it is not"
    It is measured, not estimated. Every session this project has run is
    logged with a timestamp per event; the figure above sums the gaps between
    consecutive events and **discards any gap longer than 30 minutes** on the
    assumption that I had stopped working. That makes it deliberately
    conservative.

    It counts only time spent **at the keyboard in a working session.** It does
    not count class, team meetings, reading teammates' pull requests on GitHub,
    Linear, Discord, manual browser QA, testing on a physical handset, or any
    of the thinking that happens away from the machine. The real figure is
    higher than 75.5 and I cannot measure by how much, so I am reporting the
    part I can defend rather than the part that would read better.

    Read as a weekly average it is about **10 hours**, below the 17-hour
    benchmark. Two things are worth saying plainly about that. The first is
    that the unmeasured categories above are exactly where the rest of a
    software-engineering week goes. The second is that the criterion's own top
    band does not ask for the number at all — *"does not have to exceed 17
    hours per week. What matters is that the work quality is significant"* —
    so the output below is the evidence that actually matters, and the hours
    are context for it.

    The dip in **19 – 25 Oct** is real and not hidden: that week went to a
    pull request that was blocked on a conflict and on review rather than on
    work of mine.

### Output

| | Count |
|---|---|
| Commits authored | **341** — 111 backend, 230 mobile |
| Pull requests opened | **20** — 17 merged, 1 withdrawn, 2 still open |
| Teammates' pull requests reviewed | **12**, covering **all four** teammates |
| Weeks documented to this standard | **7** |

The one withdrawn pull request is counted here on purpose. PIL-305 stopped in a
shippable state — 14 strict TDD commits, CI and SonarCloud green — and was
closed because the feature was not going to be used. Quantity that did not ship
is still quantity spent, and hiding it would make the other nineteen look
cheaper than they were.
