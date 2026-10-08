# AI literacy

I use Claude Code as a working tool on this project. What decides whether that
produces good work is not the generation. It is the **context I supply**, which
the tool has no way to obtain, and the **verification I insist on** before
accepting anything.

## What I bring that the tool cannot

Everything below originates with me, my lead dev, or my teammates. None of it is
inferable from the codebase.

| Context I provided | What it changed |
|---|---|
| **The three project documents** — IR Guidelines, PRD, SDS | Every delivery is shaped to the rubric and the product spec rather than to generic best practice. Reading the IR Guidelines myself is what caught that the security criterion is scored by *count* of OWASP items, not by naming one |
| **My lead dev's navigation rule** — in the app the bar is at the bottom, in a browser the navigation is on the left | Replaced a width-only design that was wrong in both directions. The clearest case of my context beating the tool's own answer — see below |
| **The hamburger for phone browsers** — my call on how that rule extends to a narrow window | Produced the third navigation form, `RoleNavigationDrawer` |
| **Tristan's money rule** — rupiah is whole, always rounded *down*, decided 17 Sep | Overrode the `ROUND_HALF_UP` reading implied by PIL-138's notes. Became `bulatkan_rupiah`, now shared across the backend |
| **Agreements with Pascal on PIL-223** — which fields lock, one 403 for the whole request, `email` deferred until PIL-154 merged | The tool could not have known `email` was about to become the account-linking key, so locking it then would have been wrong for a reason invisible in the code |
| **Web is Super Admin, Pengurus and Pengurus Induk only** — never Nasabah | Scoped the entire web shell, including which roles fail closed |
| **Which PRs to review, and why those** | I picked be #89 and mobile #69 because they touch code I wrote — the rupiah helper and paging. Reviewer credibility is a judgement about *me*, not about the diff |
| **Scope decisions** — one PR not three; proceed without consulting the PO; stop the emulator and use a phone | Each one closed off a path the tool was willing to keep walking down |

The pattern: the tool is good at *how*, and has no access to *why this, here,
now*. Most of the real decisions on this project were the second kind.

## How I use it

- **Drafting**, which I then read line by line — implementation, tests, commit
  bodies, PR descriptions, review comments.
- **Running the checks and reporting their output** — `flutter analyze`,
  `dart format`, the test suite, builds. The output is the evidence, not a
  summary of it.
- **Driving the TDD cycle I specify** — a failing test committed on its own,
  then the smallest change that passes it.
- **Browser QA automation** — Playwright against a running app, which produced
  the screenshots in the [PIL-293 case study](../contributions/pil-293.md).

One standing rule, which exists because of the first entry below:

!!! important "It does not get to tell me something works"
    A claim that a build passes, a test is unrelated, or a page renders has to
    arrive with the command and its output. Without that I do not accept it.

## Proof: the session history

The prompt history is retained rather than discarded. These are my actual
prompts, at the points where I rejected or redirected what I was given.

??? failure "“it's only white though?” — a working app reported from a clean log"
    It told me twice that the web app was booting correctly, based on build logs
    with no errors. The page was blank white. I sent a screenshot.

    A clean log is not a rendered page. It switched to verifying with
    screenshots of the real UI, and that habit is what later caught the
    collapsed rail having no labels.

??? failure "“the navbar should be on the bottom for mobile, left for web”"
    The most consequential one. Its own review had found that a phone in
    landscape is 844px wide and therefore lost its bottom bar — and its proposed
    fix was to add Material's window *height* class as a second dimension.

    I gave it my lead dev's rule instead: platform decides, not size. That made
    the bug disappear rather than be guarded against, and let a whole layer of
    viewport workarounds be deleted from the test suite. A domain rule from a
    human beat a technically correct elaboration from the tool.

??? failure "“wait why do we need to split the PR into three PR?”"
    It proposed splitting my work into three pull requests and implied the rubric
    required it. I had read the criterion and it says no such thing — it counts
    *meaningful* merge requests, and that qualifier exists precisely to penalise
    splitting work to inflate a count.

    Not a wrong fact: a confident reading of a document in its own favour. Harder
    to catch than a wrong number.

??? failure "“have you implemented the OWASP Top 10 in the commit message”"
    It had not. All the commits had shipped with no OWASP reference, which the
    rubric asks for in the commit body — and the same thing had already been
    missed once on PIL-168. I asked a direct question about a requirement instead
    of assuming it was handled.

??? failure "“i still don't understand about the collapsed rail part”"
    Its explanation of a bug it had found was not understandable, so I said so
    rather than nodding along. Only after the second, clearer explanation could I
    agree the fix was worth making. Approving a change I did not understand would
    have been the easier path.

??? failure "“u mean pr #56 and #59 conflicted? there's no pr #322”"
    It cited a pull request number that did not exist, mixing a Linear ticket ID
    with a GitHub PR number.

??? failure "“i closed the emulator, it took so much time”"
    It set up an Android emulator needing a 929 MB image that had not booted
    after twelve minutes. I stopped it and asked for an alternative. Judging when
    a suggested path is not worth the wait is my call.

It also corrected itself several times, kept separately because self-correction
is cheap to claim: an asserted merge conflict that `git merge-tree` disproved, a
current tool called deprecated from its version number, a screenshot offered as
proof of something it did not show, a claim that nine test failures were all
pre-existing when two were its own regression, and a test it deleted for passing
before the fix.

## Counting the corrections

Anecdotes do not show a pattern; tallying them by *cause* does. This tally is
what changed how I work, more than any single incident.

| Cause | Count | Incidents |
|---|---|---|
| Claimed verified, verification too narrow | 3 | blank page from clean logs (×2), "all nine failures pre-existing", content-cap screenshot |
| Inferred instead of checked directly | 2 | merge conflict never tested, tool called deprecated from its version number |
| Requirement known but not built into planning | 2 | OWASP missing from commits on PIL-168, then again on PIL-293 |
| Claim stronger than the evidence | 2 | PR split "required" by the rubric, Android build "blocked" |

**What the counts say.** Seven of nine sit in the top two rows, and both are the
same underlying mistake: a conclusion reached by inference when direct evidence
was one command away. That share was higher than I expected, and it is not a
knowledge problem — the checks were all cheap.

**What changed as a result**, in order of how much each has since caught:

1. **Screenshots replaced log tails** for anything user-visible.
2. **"Verified" must name its scope** — *which* files were run, which is what
   would have exposed the two untested ones.
3. **The OWASP item is chosen at slice-planning time**, not at commit time. Two
   misses of the same shape meant the remedy had to move earlier in the process
   rather than be applied harder at the end.
4. **Plans written before the work** — the
   [programming principles](../process/programming.md) page states what PIL-340
   will claim before its commits exist, so the claim can be checked against them
   instead of reverse-engineered from them.

Item 3 is still open: it has recurred once, and one successful application is
not yet evidence the fix holds.

## Why the record includes my own mistakes

Several entries above make my earlier statements look wrong too — the blocker I
reported as blocking turned out to need only a config file present, and I had
accepted the "nine pre-existing failures" claim before CI contradicted it.

That is deliberate. A log of AI use containing no disagreements is not a log of
AI use.
