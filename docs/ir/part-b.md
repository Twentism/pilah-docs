# Part B — hard skills

## B1 · Test-driven development

**Claim.** Disciplined red-green-refactor, visible in the commit history, with
positive, negative and boundary cases.

**Evidence.** [PR #70's 31 commits](https://github.com/bank-sampah-PILAH/pilah-mobile/pull/70/commits) — 13 `red`, 13 `green`, 4 `refactor`, 1 `test`.
Every cycle is two commits: the failing test, then the smallest change that
passes it. Test and implementation are never in the same commit, because if they
were, nobody could tell which came first.

Boundary coverage is deliberate, not incidental. Window width is a continuous
input, so it is partitioned and tested on-point and off-point:

| Partition | On-point | Off-point |
|---|---|---|
| compact | 599 | 600 |
| medium | 600, 839 | 599, 840 |
| expanded | 840 | 839 |

Plus degenerate input: a non-positive width returns compact rather than
throwing.

Also worth showing: one slice produced **no production code**. The keyboard
tests passed immediately because Material's `NavigationRail` already handles Tab
and Enter. I kept them as regression cover and committed them as `test(nav)`
rather than `green`, with the reason in the message — rather than invent a
widget to make the slice look productive.

See [TDD](../process/tdd.md) for the full convention and the corrections I have
taken on it.

## B2 · Programming principles

**Claim.** Named principle, pointed at the diff.

**Single Responsibility.** `RoleNavigationBar` previously held two
responsibilities: *which* destinations a role has, and *how* they are drawn.
Extracting `RoleDestinations` separates them — the data belongs to one module,
the rendering to each widget.

**Open/Closed.** The rail was added **without modifying** `RoleNavigationBar`'s
render path. That is checkable rather than asserted: the bar's `build()` output
is byte-for-byte unchanged in that commit, and the 18 existing navigation tests
passed with none of them edited.

**Strategy** and **Dependency Inversion** also apply — one pure selector with
three interchangeable renderers, and a `@visibleForTesting` platform seam
without which the whole web branch would be untestable.

The ordering is the evidence. I extracted the shared source in its own
red-green pair *before* writing the rail and the drawer, specifically so each
new form could be added additively. Both commit bodies say which principle
applies and why.

Full write-up with commit links, including what I deliberately do **not**
claim: [Programming principles](../process/programming.md).

## B3 · Development discipline

**Claim.** Descriptive Conventional Commits, PR titles in the agreed form, more
than two meaningful MRs per week.

| | |
|---|---|
| Merged PRs | 5 backend, 3 mobile |
| Open | 1 (PR #70), 31 commits |
| Commit style | `red/green/refactor(scope): subject`, body explains *why* |
| PR titles | `PIL-<n>: <short imperative>` |

Commit bodies record the commands actually run and their output, so a reviewer
can re-run them:

```text
flutter test test/design/layout/layout_breakpoint_test.dart -> 10 lulus
flutter test test/features/main/role_navigation_test.dart   -> 15 lulus, tidak ada regresi
flutter analyze lib/design/layout test/design/layout        -> No issues found
```

!!! note "On splitting PRs for the count"
    I considered splitting PIL-293 into three PRs to raise the MR count, and
    did not. The criterion counts *meaningful* MRs per week across all work, and
    splitting one coherent change to inflate a number is exactly what the
    "meaningful" qualifier excludes. One PR, 22 legible commits.

## B4 · Peer review

**Claim.** Reviews that state a positive point, a negative point and a feasible
concrete fix.

**Sprint 2 evidence:**

- [be #89](https://github.com/bank-sampah-PILAH/pilah-be/pull/89#pullrequestreview-5455262911)
  — verified the refactor really is semantics-preserving, then showed with a
  worked numeric example that the new test cannot distinguish per-step rounding
  from rounding once at the end, and proposed the specific missing case.
- [mobile #69](https://github.com/bank-sampah-PILAH/pilah-mobile/pull/69#pullrequestreview-5455267951)
  — identified that a claimed fix had already merged that morning, with file and
  line references, and challenged a `coverage:ignore` as deferring a decision
  rather than making one.

Sprint 1: be #48 and #49, both on Heraldo's work.

**Honest limitation.** This is my weakest criterion. The pattern is that I
review well when the PR touches code I wrote and less often otherwise — both
Sprint 2 reviews were chosen on that basis. That makes the reviews good and the
coverage narrow.

## B5 · Code quality

**Claim.** Linters and type checking clean on new code; no new SonarCloud
issues.

| Check | Result |
|---|---|
| `flutter analyze lib test` | No issues found |
| `dart format --set-exit-if-changed` | 512 files, 0 changed |
| SonarCloud, `feature/pil-293` | pass |
| CI full suite | 1358 pass, 1 skip, 0 fail |

Backend equivalents: `ruff`, `ruff format`, `mypy` strict, and
`makemigrations --check`.

See [Code quality](../process/quality.md), including why a green pipeline does
not prove the application starts.

## B6 · Security

**Claim.** OWASP Top 10 items named in the **commit message body**.

**Level, stated honestly.** The criterion scores by *count*: one item is level
1, **five** is level 2, and levels 3–4 need a Metasploit-style scan that this
project does not yet use. PIL-293 names A01 only, so it is level 1. The five-item
case arrives with PIL-340, where the feature touches money — planned in advance
and marked for what I wrote versus what I am citing, in
[Security](../process/security.md).

OWASP **A01 (Broken Access Control)** appears in five PIL-293 commits — both
`RoleDestinations` commits, both navigation rail commits, and the navigation
drawer. The other twenty-six have no security angle and do not claim one.

The substance: one source of truth for role-to-destination mapping, so adding a
second *or third* navigation surface cannot widen any role's menu; `forRole`
fails closed on unknown roles; rail and drawer both render nothing without a
session. All of it is pinned by tests.

Each message states that this is **defense in depth** and names where the real
control lives — server-side authorization plus the router redirect.

!!! warning "Twice-missed, now a planning step"
    On PIL-168 the OWASP note was in the PR body instead of the commit. On
    PIL-293 it was missing entirely until it was caught after the PR was open,
    then fixed by rewriting four commit messages. The remedy is to choose the
    OWASP item while planning the slices, not when writing the commit.
