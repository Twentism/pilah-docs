# IR Part C — soft skills

Organised by **week**, the same as [Part B](part-b.md). Part C is thinner per
week, so the weeks are sections here rather than separate pages — there is not
enough in any one of them to justify its own page, and splitting would pad
rather than clarify.

| Week | Dates (2026) | What it shows |
|---|---|---|
| [Sprint 1 · Week 1](#sprint-1-week-1) | 15–21 Sep | Learning decimal rounding modes properly |
| [Sprint 1 · Week 2](#sprint-1-week-2) | 22–28 Sep | Asking instead of guessing; reviewing where I add value |
| [Sprint 1 · Week 3](#sprint-1-week-3) | 29 Sep – 1 Oct | Sprint close |
| [Sprint 2 · Week 1](#sprint-2-week-1) | 6–12 Oct | Worktrees and stacked branches; Flutter web; four blockers reported |

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
[Part B · B4](b/s1-w2.md#b4--peer-review).

### Being wrong

!!! failure "I asserted a merge conflict that did not exist"
    I said two PRs conflicted. Testing it with `git merge-tree --write-tree`
    returned clean — they touched different regions of the same file. I had
    inferred the conflict from the file list instead of checking.

---

## Sprint 1 Week 3

Sprint review, retrospective, and Sprint 2 planning. No code.

The decision that mattered: web supports **Super Admin, Pengurus and Pengurus
Induk only**, never Nasabah. One sentence in planning that shaped every role
check in the next sprint.

---

## Sprint 2 Week 1

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
  properly in a PR would have duplicated someone's in-flight work.

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

## Across every week — AI literacy

I use Claude Code as a working tool, and the useful part is not the generation.
It is being a reviewer of output that is confidently wrong sometimes.

The full record, with my actual prompts at the points where I rejected or
redirected what I was given, is **[B7 · AI literacy](ai-literacy.md)**.
