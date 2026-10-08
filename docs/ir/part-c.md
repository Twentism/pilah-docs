# Part C — soft skills

## Learning beyond the classroom

TDD and refactoring were taught in class. These were not, and each was learned
because a specific problem needed it.

### Git worktrees and stacked branches

PIL-293 depended on a fix that had not merged yet: staging's Flutter web build
compiled and then died with `FirebaseOptions cannot be null`. Waiting would have
cost days, so the branch was based on Tristan's unmerged branch instead.

That required things I had not used before — worktrees so several branches can
be checked out at once, `git rebase --onto` to re-parent a stack when the base
moves, and `--force-with-lease` to push a rewritten branch without clobbering
anyone.

It got exercised twice. Tristan force-pushed his base, and then his work
squash-merged into staging — which changes the lineage while leaving the tree
identical. Confirming that before rebasing is what made it safe:

```bash
git merge-base --is-ancestor origin/feature/pil-334 origin/staging   # no
git diff --stat origin/feature/pil-334 origin/staging                # empty
git rebase --onto origin/staging origin/feature/pil-334 feature/pil-293
```

Different lineage, identical content — so a 21-commit rebase with zero
conflicts. The lesson I did not have before: a squash merge makes `git log`
disagree with `git diff`, and the diff is the one that decides whether a rebase
will hurt.

### Decimal rounding modes

PIL-168 looked like a one-line rounding change and was not. `ROUND_HALF_UP`,
`ROUND_DOWN` and `ROUND_FLOOR` differ in ways that matter for money:
`ROUND_DOWN` truncates toward zero, `ROUND_FLOOR` toward negative infinity, and
they diverge on negative values. Choosing one meant understanding what a
fraction of a rupiah *means* in a system migrating legacy balances that carry
sen, not just which function name to type.

It paid off later in a review: because I knew the mode mattered, I checked
Vegard's refactor by comparing the helper's actual quantize call against what it
replaced, rather than trusting the PR description.

### Flutter web as a compile target

Not a separate app — the same Dart compiled to JavaScript, painted to a canvas.
Consequences I had to learn the hard way: browser automation snapshots come back
empty because there is no DOM, so clicking has to be done by coordinate; and
`kReleaseMode` compiles the demo login out, which makes release-build QA
impossible without a real OAuth credential.

## Working with the team

**Asking instead of guessing.** On PIL-223 the `email` field could not be locked
yet, because the PR that makes it the account-linking key had not merged. I
raised it with Pascal, we agreed to defer it, and it landed in PIL-288 — rather
than implementing a lock whose rationale did not exist yet.

**Reporting blockers instead of routing around them.** Three on PIL-293: a
missing `google-services.json` blocking the APK check, no web OAuth client ID
blocking release QA, and a DI regeneration defect in somebody else's merged
code. I reported all three on the PR and fixed none of them, because none were
mine to fix. The DI one is the interesting case: it is invisible to CI, so
saying nothing would have left a trap for whoever ran `build_runner` next.

**Reviewing where I can actually add value.** Both Sprint 2 reviews were chosen
because the PRs touched code I had written — the rupiah helper from PIL-168 and
paging behaviour from PIL-214. A review from someone who knows the constraints
is worth more than a broad skim.

## Being wrong

The part I would most want a reader to check, because it is where documentation
is usually quietest.

!!! failure "I reported a working app from a clean log"
    Twice I said the web app booted, based on build logs with no errors. The
    page was blank white. A clean log is not a rendered page. I now verify with
    a screenshot of the actual UI.

!!! failure "I claimed nine failures were pre-existing"
    Seven were. Two were my own regression, in files my verification had not
    touched — and I had said "verified". CI contradicted me and was right. The
    fix was narrow: a claim to have verified something has to name what was
    verified.

!!! failure "I asserted a merge conflict that did not exist"
    I told the user two PRs conflicted. Testing it with
    `git merge-tree --write-tree` returned clean — they touched different
    regions of the same file. I had inferred the conflict from the file list
    instead of checking.

!!! failure "I called a current tool deprecated"
    Dismissed a CLI as abandoned because its version number was `0.1.22`. That
    was its own versioning scheme; the tool was current.

!!! failure "I missed the OWASP requirement twice"
    PIL-168 had it in the PR body instead of the commit. PIL-293 had it nowhere
    until it was caught after the PR was green. Both times the remedy was known
    and I had not built it into how I plan work.

What connects these: every one came from inferring a conclusion that was cheap
to check directly. Logs instead of the screen, a file list instead of a merge
test, a version number instead of the docs. The habit I am building is to check
the thing itself when checking is cheap — and when it is not cheap, to say which
one I did.

## AI literacy

I use Claude Code as a working tool, and the useful part is not the generation —
it is being a reviewer of output that is confidently wrong sometimes. Several
corrections above came from me pushing back: the non-existent merge conflict,
the "deprecated" tool, an overstated rationale for splitting PRs into three. The
prompt history is kept for exactly this reason, so the places where a claim was
challenged are visible rather than smoothed over.
