# IR Part B — hard skills

Organised by **week**, because that is how the work happened and how it gets
assessed. Each week carries its own B1–B7 evidence; nothing is claimed twice,
and nothing is claimed before it exists.

## The weeks

| Week | Dates (2026) | What shipped | Criteria with evidence |
|---|---|---|---|
| [Sprint 1 · Week 1](b/s1-w1.md) | 15–21 Sep | Whole-rupiah money rules, deposit input limits | B1 B2 B3 B5 B6 |
| [Sprint 1 · Week 2](b/s1-w2.md) | 22–28 Sep | Nasabah paging, profile-edit permissions, account linking | B1 B2 B3 B4 B5 B6 B7 |
| [Sprint 1 · Week 3](b/s1-w3.md) | 29 Sep – 1 Oct | Sprint close — no commits of mine | — |
| [Sprint 2 · Week 1](b/s2-w1.md) | 6–12 Oct | Responsive web shell (PIL-293), setoran on web (PIL-340) | B1 B2 B3 B4 B5 B6 B7 |

Sprint 2 runs to 22 Oct, so Weeks 2 and 3 do not exist yet. They are not listed
until they have something in them.

## Deep dives

Two tickets were large enough to deserve their own write-up rather than a
paragraph inside a week:

<div class="grid cards" markdown>

- :material-view-dashboard-outline: **[PIL-293 — the responsive web shell](../contributions/pil-293.md)**
  Platform decides the navigation family, width decides how much of it fits.

- :material-cash-register: **[PIL-340 — recording a setoran on the web](../contributions/pil-340.md)**
  Six slices, three bugs found by tests, two refuted design claims.

- :material-robot-outline: **[B7 — AI literacy](ai-literacy.md)**
  The full session record: where the tool was wrong and how it was caught.

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
[Sprint 1 · Week 2](b/s1-w2.md#b4--peer-review) for a reviewer who did exactly
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
[Sprint 2 · Week 1](b/s2-w1.md#test-doubles-what-is-faked-and-what-it-costs).

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
    The best week so far names **three** distinct items (Sprint 2 · Week 1),
    which is level 1, not level 2. Levels 3 and 4 need a scanning tool nothing
    in this project currently uses. I would rather report three real items than
    pad to five.

### Secrets handling

Not an OWASP item as such, but it is where mistakes are cheapest to make:

- `.env`, Firebase credentials and `google-services.json` are never committed.
  `.env` is a declared Flutter asset, so builds fail without it — which makes
  the temptation to commit it real.
- CI materialises `google-services.json` from a repository secret. When I could
  not build a debug APK locally without it, I reported the blocker rather than
  working around it.
- Local QA runs with a permissive CORS setting and a fake-Google-token flag.
  Both are development-only, documented as such, and the CORS patch described
  in Sprint 2 · Week 1 is deliberately left **uncommitted**.
