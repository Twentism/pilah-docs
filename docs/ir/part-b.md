# IR Part B — hard skills

Organised by **week**, because that is how the work happened and how it gets
assessed. Each week carries its own B1–B7 evidence; nothing is claimed twice,
and nothing is claimed before it exists.

## The weeks

| Week | Dates (2026) | What shipped | Criteria with evidence |
|---|---|---|---|
| [Sprint 1 · Week 1](b/s1-w1.md) | 15–21 Sep | Whole-rupiah money rules, deposit input limits | B1 B2 B3 B5 B6 B7 |
| [Sprint 1 · Week 2](b/s1-w2.md) | 22–28 Sep | Nasabah paging, profile-edit permissions, account linking | B1 B2 B3 B4 B5 B6 B7 |
| [Sprint 1 · Week 3](b/s1-w3.md) | 6–12 Oct | Responsive web shell (PIL-293), setoran on web (PIL-340) | B1 B2 B3 B4 B5 B6 B7 |
| [Sprint 2 · Week 1](b/s2-w1.md) | 12–16 Oct | Cross-origin custom headers in the backend (PIL-354), Pengurus schedule on the web (PIL-339) | B1 B2 B3 B4 B5 B6 B7 |
| [Sprint 2 · Week 2](b/s2-w2.md) | 13–19 Oct | Jenis sampah and harga on the web (PIL-338), and the panel-breakpoint defect PIL-339 shipped | B1 B2 B3 B5 B7 |
| [Sprint 2 · Week 3](b/s2-w3.md) | 20–26 Oct | An unauthenticated reflected XSS fixed, and a Content-Security-Policy per response context — both merged | B1 B2 B3 B4 B5 B6 B7 |

Sprint 2 closed on 26 Oct and Sprint 3 begins 27 Oct, so later weeks do not
exist yet. They are not listed until they have something in them.

!!! info "Two of the week headings are reporting slots, not claim windows"
    The claim windows are now fixed: Sprint 2 ran **6–12 Oct** (Week 1),
    **13–19 Oct** (Week 2) and **20–26 Oct** (Week 3). The last two pages sit
    exactly in theirs.

    The two earlier pages do not, and are left as submitted rather than
    retrofitted: **Sprint 1 · Week 3** reports work done 6–12 Oct, and
    **Sprint 2 · Week 1** reports work done 12–16 Oct. Each carries a note
    saying so.

    Every date in the table is the real date of the real work. Where a slot
    label and a claim window disagree, the dates are the thing to trust, and
    nothing is claimed on two pages.

Sprint 1 · Week 1 claims no B4, because no peer review happened in it, and
Sprint 2 · Week 2 claims none because its two reviews are reported on the
adjacent page. Sprint 2 · Week 2 also claims no B6: PIL-338 had no honest OWASP
claim to make. An empty criterion is left empty rather than filled with an
affirmative comment.

## Deep dives

Two tickets were large enough to deserve their own write-up rather than a
paragraph inside a week:

<div class="grid cards" markdown>

- :material-view-dashboard-outline: **[PIL-293 — the responsive web shell](../contributions/pil-293.md)**
  Platform decides the navigation family, width decides how much of it fits.

- :material-cash-register: **[PIL-340 — recording a setoran on the web](../contributions/pil-340.md)**
  Six slices, three bugs found by tests, two refuted design claims.

</div>

## Standing conventions

These do not belong to a single week — they are how every week is worked.

### The commit convention

The red-green-refactor cycle is visible in the git history, because that is the
only way a claim to practise TDD can be checked:

| Prefix | Means |
|---|---|
| `red(scope)` | A failing test for **one** behaviour. Run, and seen to fail. |
| `green(scope)` | The smallest change that makes it pass. |
| `refactor(scope)` | Real cleanup, no behaviour change. |

Two rules I hold to:

- **The test and the implementation are never in the same commit.** If they
  are, nobody can tell whether the test was written first.
- **One behaviour per cycle.** Bundling slices makes the history unreadable and
  hides which test drove which line.

Commit bodies record the commands actually run and their output, so a reviewer
can re-run them rather than take my word:

```text
flutter test test/design/layout/layout_breakpoint_test.dart -> 10 lulus
flutter test test/features/main/role_navigation_test.dart   -> 15 lulus, tidak ada regresi
flutter analyze lib/design/layout test/design/layout        -> No issues found
```

### How I find corner cases

Not by intuition — by asking three questions of each input:

1. **Where does the answer change?** Every `if` and every comparison is a
   boundary. Test the last value on each side, not a comfortable value in the
   middle.
2. **What is the smallest or emptiest valid input?** Zero weight, an empty
   destination list, a width of `0` before the first frame has measured the
   window.
3. **What does the type allow that the domain does not?** `double` permits a
   negative width; the domain does not. That produced a deliberate decision —
   return the narrowest form rather than throw, because the caller should not
   have to guard a transient startup state.

### Proving a test actually guards the line

A test that passes is not evidence that it would fail. When I add a test to pin
existing behaviour, I break the line on purpose and watch it fail, then restore
it. If it does not fail, it was not guarding anything.

I look for the same thing when reviewing — see
[Sprint 1 · Week 2](b/s1-w2.md#b4-peer-review) for a reviewer who did exactly
that to my own code.

### The rule for test doubles

1. **No double** when the unit has no collaborators.
2. **Stub** by default, as low as possible — fake the socket, keep the app.
3. **Mock** only when there is no observable result: a side effect that leaves
   the app, or a state the stub physically cannot produce.

The failure mode this avoids is a suite that mocks the layer directly beneath
the one under test. It passes forever, breaks on every refactor, and never
catches an integration bug. Worked through in full — including the practical
difference between the two, with line-level references — in
[Sprint 1 · Week 3](b/s1-w3.md#test-doubles-what-is-faked-and-what-it-costs).

Rule 1 is not a formality. [Sprint 2 · Week 1](b/s2-w1.md#test-doubles-none-and-that-is-the-honest-answer)
uses **no double at all**: the defect was in the configuration of a middleware,
so a test that faked that middleware could not have caught it.

### Gates run before every push

**Backend — `pilah-be`**

```bash
ruff check . && ruff format --check .
mypy api apps config shared_kernel tests      # strict
python manage.py makemigrations --check --dry-run
python manage.py test
```

**Mobile — `pilah-mobile`**

```bash
flutter analyze lib test                                     # --fatal-infos in CI
dart format --output=none --set-exit-if-changed lib test codegen
flutter test --coverage
flutter build web --release -t lib/main_development.dart
```

`makemigrations --check` is there because a model change without a migration
passes every other check and then breaks the next person to pull.

### What the security criterion actually asks

Worth stating plainly, because I had it wrong at first. B6 is scored by **how
many** of the OWASP Top 10 the code demonstrably prevents, named in the commit
message body:

| Level | Requirement |
|---|---|
| 1 | Code preventing **1** of the Top 10, named in the commit message |
| 2 | Code preventing **at least 5** of the 10, named in the commit messages |
| 3 | A scan with Metasploit or similar, explained, with a patch plan |
| 4 | Penetration testing with video evidence |

Note the wording: *"has shown some/all of the code that **already** prevents"*.
Pointing at existing protections counts, not only at code written this week —
so each week marks what I **wrote** against what I am **citing**.

!!! warning "The running total, stated honestly"
    Across every week the distinct items named in commit bodies are **A01,
    A03, A04 and A05** — **four**, which is still level 1, not level 2.

    **A03 (Injection / XSS) is new in [Sprint 2 · Week 3](b/s2-w3.md#b6-security)**
    and is the first genuinely new item since Sprint 1: an unauthenticated
    reflected XSS in the OAuth popup, fixed across three `green` commits, one
    per output context.

    PIL-354 did not add a fourth. It moved **A05 from cited to written**: the
    CORS gap found in Sprint 1 · Week 3 is fixed in code in
    [Sprint 2 · Week 1](b/s2-w1.md#b6-security). A better claim on the same
    item, not a bigger number.

    Level 3 is a different axis rather than more of the same — it asks for a
    **scan**, explained, with a patch plan, and it is **met**. That arrived in
    [Sprint 2 · Week 1](b/s2-w1.md#b6-security): an authenticated OWASP ZAP
    scan of the local backend, each alert triaged into real or artefact, and a
    patch plan for the one real finding. The plan was then implemented and
    merged in
    [Sprint 2 · Week 3](b/s2-w3.md#the-patch-plan-from-week-1-now-implemented-and-merged),
    which level 3 does not even require.

    **Level 4 is out of reach, and not for want of a re-scan.** It asks for
    penetration testing *with a video of the process*. There is no video, so
    level 3 is the ceiling. The re-scan matters for the Part A Secure
    Programming row, which wants the before **and after** of code fixed on the
    basis of a screening — see
    [Sprint 2 · Week 3](b/s2-w3.md#where-this-week-lands-in-part-a).

    I would still rather report four real items and a scan than pad the count
    to five.

### B7 · how I use AI

Each week's B7 section carries that week's own evidence — the context I
supplied, and the prompts where I rejected or redirected what I was given. What
cannot live in a single week is the **tally across all of them**, which is
below, because a pattern is only visible once the weeks are added up.

**How I use it.** Drafting, which I then read line by line — implementation,
tests, commit bodies, PR descriptions, review comments. Running the checks and
reporting their output: `flutter analyze`, `dart format`, the suites, the
builds, where the output is the evidence rather than a summary of it. Driving
the TDD cycle I specify, a failing test committed on its own and then the
smallest change that passes it. And browser QA automation against a running
app, which produced the screenshots in the
[PIL-293 case study](../contributions/pil-293.md).

One standing rule, which exists because of the blank-page incident in
[Sprint 1 · Week 3](b/s1-w3.md#b7-ai-literacy):

!!! important "It does not get to tell me something works"
    A claim that a build passes, a test is unrelated, or a page renders has to
    arrive with the command and its output. Without that I do not accept it.

**Counting the corrections.** Anecdotes do not show a pattern; tallying them by
*cause* does. This tally changed how I work more than any single incident did.

| Cause | Count | Incidents |
|---|---|---|
| Claimed verified, verification too narrow | 5 | blank page from clean logs (×2), "all nine failures pre-existing", content-cap screenshot, `collectstatic` run locally then called green |
| A test that passed without testing anything | 4 | `</script>` slicing that hid the injection, `DEBUG` compared three wrong ways, a test skipped in CI only, an escape map that was a no-op |
| Inferred instead of checked directly | 4 | merge conflict never tested, tool called deprecated from its version number, `DJANGO_DEBUG` blamed before reading `ci.yml`, a reviewer's suggested fix applied before being run |
| Requirement known but not built into planning | 3 | OWASP missing from commits on PIL-168, then again on PIL-293, then PIL-338's "no claim here" never written down |
| Claim stronger than the evidence | 3 | PR split "required" by the rubric, Android build "blocked", a `refactor` commit claiming no behaviour change while carrying one |

Nineteen incidents, and the shape has shifted. The top two rows are **nine of
them**, and they are the same underlying mistake in two costumes: a conclusion
reached without the one check that would have settled it. Row 1 is believing my
own environment; row 2 is believing my own test.

Row 2 is new, and it only became visible once there were four of them. A test
that passes is not evidence that it would fail, and four times this project has
produced a green assertion over code that was wrong — twice because the
technique for reading the evidence excluded it, once because the setting being
compared was not the setting in force, once because the test never ran at all.
That is what moved manual mutation testing from something I did occasionally to
something every guard added after green now gets.

**What changed as a result**, in order of how much each has since caught:

1. **Screenshots replaced log tails** for anything user-visible.
2. **"Verified" must name its scope** — *which* files were run, which is what
   would have exposed the two untested ones.
3. **The OWASP item is chosen at slice-planning time**, not at commit time. Two
   misses of the same shape meant the remedy had to move earlier in the process
   rather than be applied harder at the end.
4. **Plans written before the work** — the
   [programming principles](b/s1-w3.md#b2-programming-principles) page states
   what PIL-340 will claim before its commits exist, so the claim can be
   checked against them instead of reverse-engineered from them.

Item 3 is still open, and the record got worse before it got better: PIL-338's
twelve commits carried no OWASP line at all. The honest reading is that a
layout-and-lifecycle ticket had no claim to make — but "there was none to make"
has to be *stated at planning time*, which is the discipline the remedy was
supposed to install. The security work the following week names its items in
every `green` commit, so the remedy works when the slice has a claim; it is the
decision that there is none which is still not being recorded. The two weeks
sit side by side in [Sprint 2 · Week 2](b/s2-w2.md#b6-security) and
[Sprint 2 · Week 3](b/s2-w3.md#b6-security), which is the clearest evidence
either way.

A fifth change, from [Sprint 2 · Week 3](b/s2-w3.md#b7-ai-literacy):

5. **Every guard added after green is proved by breaking what it protects.**
   Six such mutations in that week alone. This is the direct consequence of row
   2 of the tally above — it exists because four tests passed over broken code.

**Why the record includes my own mistakes.** Several entries make my own
earlier statements look wrong too — the Android blocker I reported as blocking
turned out to need only a config file present, and I had accepted the "nine
pre-existing failures" claim before CI contradicted it. That is deliberate. A
log of AI use containing no disagreements is not a log of AI use.

### Secrets handling

Not an OWASP item as such, but it is where mistakes are cheapest to make:

- `.env`, Firebase credentials and `google-services.json` are never committed.
  `.env` is a declared Flutter asset, so builds fail without it — which makes
  the temptation to commit it real.
- CI materialises `google-services.json` from a repository secret. When I could
  not build a debug APK locally without it, I reported the blocker rather than
  working around it.
- Local QA runs with a permissive CORS setting and a fake-Google-token flag.
  Both are development-only and documented as such. The CORS patch described in
  Sprint 1 · Week 3 was deliberately left **uncommitted** at the time, because
  the team said the gap was already being handled. Once that was cleared it was
  fixed properly in the backend instead — PIL-354,
  [be #97](https://github.com/bank-sampah-PILAH/pilah-be/pull/97) — so no local
  patch is carried any more.
