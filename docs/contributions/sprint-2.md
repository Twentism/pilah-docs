# Sprint 2 — 6 to 22 Oct 2026

Sprint 2 changed direction: the team is building a **Flutter web dashboard for
Pengurus**. Not a separate web app — the same Dart codebase compiled to
JavaScript, which is why it is a `pilah-mobile` ticket despite being a web
feature.

Web is scoped to **Super Admin, Pengurus and Pengurus Induk only**. Never
Nasabah. Members stay on the phone app.

## My ticket: PIL-293

Responsive layout and the web navigation shell. Full write-up in the
[PIL-293 case study](pil-293.md) — it is the most involved thing I have built
on this project and worth reading as one piece.

Short version: the app had exactly one navigation form, a bottom bar, which is
right on a phone and wrong on a 1440px monitor. PIL-293 adds window-width
classes, a navigation rail for wide windows, a content width cap, and two
layout primitives the other web tickets will reuse.

<span class="chip chip-open">open</span>
[PR #70](https://github.com/bank-sampah-PILAH/pilah-mobile/pull/70) ·
22 commits · base `staging`

## What made this sprint harder than Sprint 1

**The web build did not boot.** Before any of my work could be verified in a
browser, staging's Flutter web build compiled fine and then died at runtime
with `FirebaseOptions cannot be null when creating the default app`. Firebase
has no web-supported flow in this app, so initialising it on web was simply
wrong. That was Tristan's fix, and my ticket had to be based on his branch
before it merged — which meant working on a stacked branch and rebasing twice
when his base moved.

**Release-build QA is impossible without a credential.** The demo login button
is compiled out of release builds (`supportsDemoLogin => !kReleaseMode && ...`),
so verifying a release web build requires a real Google OAuth client ID for
web. I do not have one. This blocks every web ticket, not just mine, and I
raised it rather than quietly skipping the check.

**Flutter web renders to canvas.** Browser automation snapshots come back empty
because there is no DOM to inspect — the whole UI is painted. Screenshots are
the only reliable verification, and clicking has to be done by coordinate. I
learned this the slow way, after twice reporting "the app boots" from reading
log tails while the page was actually blank.

!!! danger "The mistake worth naming"
    Twice I told the user the app was working based on build logs showing no
    errors. The page was white. A clean log is not a rendered page. Since then
    I verify with a screenshot of the actual UI, never a log tail. It is in
    here because it changed how I work, not because it is flattering.

## A defect I found and reported but did not fix

`AuthenticationBloc` has an optional named parameter `isWebOverride` used as a
test seam. `injectable` reads constructor parameters, so regenerating the DI
config emits `isWebOverride: gh<bool>()` — and the app then dies at startup
with `Bad state: GetIt: Object/factory with type bool is not registered`.

What makes it easy to miss: CI *does* run `build_runner`, so the committed
generated file is not what CI uses — but CI only analyses, tests and builds. It
never runs the app, so GetIt resolution is never exercised. Analyze and 1338
tests pass on the broken config.

I reported it on my PR and left it alone. It is not PIL-293's scope, and the
fix belongs with the bloc: a test seam should not be a constructor parameter
that `injectable` can see.

## Next

PIL-340, once PIL-293 is through review.
