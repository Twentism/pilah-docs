# Peer review

A useful review comment states a positive point, a negative point, **and** a
feasible concrete fix. A comment that only approves is not a review.

This is the part of my work I have had to deliberately improve.

## be #89 — rupiah rounding (Vegard)

[Review](https://github.com/bank-sampah-PILAH/pilah-be/pull/89#pullrequestreview-5455262911)

This PR refactors the pencairan paths onto `bulatkan_rupiah` — the helper I
wrote in PIL-168 — so I was well placed to check it.

**Positive, and verified rather than assumed.** I checked the substitution
myself: `bulatkan_rupiah` is `quantize(RUPIAH, rounding=ROUND_DOWN)` with
`RUPIAH = Decimal(1)`, exactly what both call sites spelled out. The
"no behaviour change" claim holds.

**The gap.** `putar` rounds *inside* a loop, once per pencairan, accumulating
through `saldo_berjalan`. The new test exercises a **single** pencairan — and
with one rounding step you cannot distinguish per-step rounding from rounding
once at the end. Both give 315600. So the test pins that *a* rounding happens on
that line, but not that it happens *per step*. Hoisting the rounding out of the
loop would keep the test green.

The two only diverge when sen re-enters between two rounding steps:

```text
setoran   +10.9  -> 10.9
pencairan   1    ->  9.9 -> 9     (rounded)
setoran    +0.9  ->  9.9
pencairan   1    ->  8.9 -> 8     (rounded)      final = 8

rounding once at the end: 10.9 - 1 + 0.9 - 1 = 9.8 -> 9
```

**Concrete fix.** Add a case with two pencairan and a sen-bearing setoran
between them, asserting the intermediate snapshot as well as the final saldo.

**And a caveat on my own point.** This only bites if a setoran can carry sen. In
PILAH 2.0 it cannot — the subtotal goes through `bulatkan_rupiah`. But the
premise of the whole PR is legacy PILAH 1.0 rows that do. If legacy setoran are
always whole, the test is unnecessary, and one line in the loop saying so would
save the next reader the question.

## mobile #69 — activity filters and pagination (Rifqi)

[Review](https://github.com/bank-sampah-PILAH/pilah-mobile/pull/69#pullrequestreview-5455267951)

**Positive.** The paging handles the three things paging usually gets wrong:
rejecting stale responses, deduplicating appended rows, and keeping existing
rows when the *next* page fails. I hit all three on PIL-214, so it was good to
see them handled deliberately rather than discovered later.

**Two findings.**

*The web-startup bullet is no longer a fix.* It describes skipping Firebase on
web and using the web OAuth client ID — but that merged into staging earlier the
same day in #60, and is now inline at `main_development.dart:32,37-38`. After a
rebase, the new `initializePlatformServices()` is a *refactor* of merged code,
which is an improvement but not a bug fix. Concretely: rebase, make the
`main_*.dart` entrypoints call the helper so the policy exists once rather than
inline *and* extracted, and re-word the bullet so reviewers do not hunt for a
bug that is gone.

*`coverage:ignore` on a navigation guard.* A dead branch in `main_page.dart` is
wrapped in `coverage:ignore-start/end` with the note "Dead today". If it is
genuinely unreachable, delete it. If it guards a state that could return, it
deserves a test. The suppression is the one option that leaves the question
open.

**Scope honesty.** 68 files and +2602/−1013 in one PR is more than one pass can
cover. I said explicitly which parts I had read properly and which I had not, so
the review is not mistaken for a sign-off on the rest.

## be #48 and #49 — modular monolith and contract tests (Heraldo)

Earlier reviews, Sprint 1. On #48 I flagged a rebase risk landing on code I had
written: the branch predated four merged PRs, and two pure rule modules
(`api/keanggotaan.py`, `api/kalkulasi.py`) did not exist on it at all. On #49 I
listed the specific nasabah rules the new contract tests did not yet cover —
`punya_akun` on list and detail, the 403 for editing a linked nasabah, the 403
for an inactive one — each tied to the ticket that introduced it.

## What I am working on

Reviewing is my weakest area, and the pattern is visible: I review well when the
PR touches code I wrote, and less often otherwise. The two Sprint 2 reviews
above were both chosen on that basis deliberately — but choosing only familiar
PRs is a limit, not a strategy.
