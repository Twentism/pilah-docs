# AI literacy — session history

I use Claude Code as a working tool on this project. The interesting evidence
is not that it writes code; it is the record of **where its output was wrong
and how that was caught**. That record is kept deliberately, because a
transcript showing only successes would be evidence of nothing.

What follows is drawn from the working sessions on PIL-293. Quotes are my
actual prompts.

## How the work is divided

| I decide | The tool does |
|---|---|
| What the ticket means, and its scope | Drafts implementation and tests |
| Which design rule applies | Proposes options with trade-offs |
| Whether a claim is actually verified | Runs the commands and reports output |
| What goes in a commit or a review | Writes the prose for me to check |

The rule I hold to: **it does not get to tell me something works.** A claim
that a build passes or a test is unrelated has to come with the command and its
output, or I do not accept it. That rule exists because of the first item
below.

## Where I corrected it

### "it's only white though?"

It reported twice that the web app was booting correctly, based on build logs
containing no errors. The page was blank white. I sent a screenshot.

A clean log is not a rendered page. After this it switched to verifying with
screenshots of the actual UI, and the habit carried into the rest of the
ticket — the final QA on PIL-293 is four screenshots of a running app, not a
log tail.

This is the correction I value most, because the failure mode was *confident
and specific* rather than vague.

### "u mean pr #56 and #59 conflicted? there's no pr #322"

It referred to a pull request number that did not exist, mixing a Linear ticket
ID (PIL-322) with a GitHub PR number. Small, but the kind of error that wastes
a reviewer's time if it reaches a PR description.

### "wait why do we need to split the PR into three PR?"

It proposed splitting my work into three pull requests and implied the grading
rubric required it. I pushed back, because I had read the criterion and it says
no such thing.

It was overstating. The criterion counts *meaningful* merge requests per week
across all work, and the "meaningful" qualifier is specifically there to
penalise splitting work to inflate a count. We shipped one PR with 31 legible
commits instead.

Worth noting what the error was: not a wrong fact, but a **confident reading of
a document in its favour**. Those are harder to catch than a wrong PR number.

### "so far I didn't see any of those as requirements"

Same session, same shape. I asked it to justify a claim against the actual
source rather than from memory, and the justification did not survive.

### "i still don't understand about the collapsed rail part"

Its explanation of a bug it had found was not understandable, so I said so
rather than nodding along. The second explanation — two widths of the same
widget, and a separate setting controlling whether captions appear — was clear,
and only then could I agree it was worth fixing.

If I had not asked, I would have approved a change I did not understand.

### "the navbar should be on the bottom for mobile, left for web"

The most consequential one. It had built the navigation decision entirely from
window width, and its own review found a bug: a phone in landscape is 844px
wide, so it lost its bottom bar.

Its proposed fix was to add Material's window *height* class as a second
dimension. I passed on my lead dev's rule instead — platform decides, not size.

That was the better answer, and it made the bug disappear rather than be
guarded against. The whole layer of viewport workarounds in the test suite was
deleted as a result. A domain rule from a human beat a technically correct
elaboration from the tool.

### "have you implemented the OWASP Top 10 in the commit message"

It had not. All 31 commits had shipped with no OWASP reference, which a
specific criterion asks for in the commit body — and the same thing had already
been missed once, on PIL-168.

I asked a direct question about a requirement rather than assuming it had been
handled. It had not been.

### "i closed the emulator, it took so much time"

It set up an Android emulator that needed a 929 MB image and had not booted
after twelve minutes. I stopped it and asked for an alternative. Knowing when a
suggested path is not worth the wait is my call, not the tool's.

## Where it corrected itself

Kept here because self-correction is cheap to claim and easy to check.

- **"All nine failures are pre-existing."** It said this, and said it had
  verified it. CI then failed two tests. Seven were pre-existing; two were its
  own regression, in files its check had not covered. The claim was not false so
  much as **overreaching** — a check across three files cannot speak for a
  suite.
- **An asserted merge conflict that did not exist.** It said two PRs conflicted.
  Testing with `git merge-tree --write-tree` came back clean: they touched
  different regions of the same file. It had inferred the conflict from a file
  list.
- **A "deprecated" tool that was current.** Dismissed a CLI because its version
  read `0.1.22`. That was the tool's own versioning scheme.
- **A screenshot that did not show what it claimed.** It offered a 1440×900
  capture as proof the content-width cap worked. At that width the cap is a
  no-op (1440 − 256 rail = 1184, under the 1200 limit). The screenshot was real;
  the reading of it was wrong.
- **A test that passed before the fix.** While fixing the unlabelled rail it
  wrote a test for label text that passed against the broken code, because the
  text is in the widget tree but unpainted. It deleted the test rather than keep
  a false guarantee.

## Counting the corrections

Anecdotes do not show a pattern. Tallying the same corrections by *cause*
does, and the tally is what changed how I work rather than any single
incident.

| Cause | Count | Incidents |
|---|---|---|
| Claimed verified, verification too narrow | 3 | blank page from clean logs (×2), "all nine failures pre-existing", content-cap screenshot |
| Inferred instead of checked directly | 2 | merge conflict never tested, tool called deprecated from its version number |
| Requirement known but not built into planning | 2 | OWASP missing from commits on PIL-168, then again on PIL-293 |
| Claim stronger than the evidence | 2 | PR split "required" by the rubric, Android build "blocked" |

**What the counts say.** Seven of nine sit in the top two rows, and both rows
are the same underlying mistake: a conclusion reached by inference when direct
evidence was one command away. That is a bigger share than I expected, and it
is not a knowledge problem — the checks were all cheap.

**What changed as a result**, in order of how much each has caught:

1. **Screenshots replaced log tails** for anything user-visible. Directly
   caused by the blank-page incidents; it is also what caught the collapsed
   rail having no labels.
2. **"Verified" must name its scope.** After the nine-failures miss, a claim
   now says *which* files were run, which is what would have exposed that two
   of them were untested.
3. **The OWASP item is chosen at slice-planning time**, not at commit time.
   Two misses with the same shape meant the remedy had to move earlier in the
   process, not be applied harder at the end.
4. **Plans written before the work, not after.** The
   [programming principles](../process/programming.md) page states what PIL-340
   will claim *before* its commits exist, so the claim can be checked against
   them rather than reverse-engineered from them.

Row three is the one still open: it has recurred once already, and one
successful application is not yet evidence the fix holds.

## What I take from this

Every one of these came from a conclusion that was **cheap to check directly
and was inferred instead** — logs instead of the screen, a file list instead of
a merge test, a version number instead of the documentation, a remembered
rubric instead of the rubric.

So the working rule is not "distrust the tool". It is narrower and more useful:

> When checking is cheap, check the thing itself. When it is not cheap, say
> which check was actually run.

That is also why the commit bodies in this project record the exact commands
and their output. It is not ceremony — it is the only form in which "this
works" means anything.

## On keeping the record

The prompt history is retained rather than discarded at the end of a session,
and the corrections above are reproduced here rather than summarised away.
Several of them make my own earlier statements look wrong too — the blocker I
reported as blocking, for instance, turned out to need only a config file
present. That is the point. A log of AI use that contains no disagreements is
not a log of AI use; it is marketing.
