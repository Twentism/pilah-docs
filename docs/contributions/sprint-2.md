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
right on a phone and wrong on a 1440px monitor. PIL-293 makes the platform
decide — bottom bar in the app, left-hand navigation in a browser — adds a
navigation rail and a drawer for narrow browsers, caps the content width, and
ships three layout primitives the other web tickets were expected to reuse.
Two of those three turned out not to fit — see PIL-340 below.

<span class="chip chip-merged">merged</span>
[PR #70](https://github.com/bank-sampah-PILAH/pilah-mobile/pull/70) ·
35 commits · merged into `staging` · approved by @HeraldoArman · SonarCloud 0 issues

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

## A rule that arrived mid-ticket

My first implementation chose the navigation form from window width alone. My
lead dev then set the rule out plainly: **in the app the navigation is at the
bottom, in a browser it is on the left.**

That turned out to matter more than a styling preference. Width alone was wrong
in both directions — a native phone in landscape is 844px wide and lost its
bottom bar, and a browser narrowed below 600 grew one. Switching the decision
to `kIsWeb` fixed both, and let me delete a whole layer of viewport workarounds
from the test suite that had only existed to paper over the wrong decision.

Narrow browsers needed a third form, so they now get a menu button and a
drawer. Details in the [case study](pil-293.md).

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

## My second ticket: PIL-340

Recording a setoran on the web — the one screen in the dashboard that writes.
Full write-up in the [PIL-340 case study](pil-340.md).

Short version: six slices built on PIL-293's shell. Validation moved out of the
487-line page widget into a value object, the form became two columns on wide
windows, the nasabah picker and the success confirmation became dialogs in a
browser while staying bottom sheets on a phone, the form now freezes while a
save is in flight, and the whole thing can be driven from the keyboard.

<span class="chip chip-open">open</span>
[PR #72](https://github.com/bank-sampah-PILAH/pilah-mobile/pull/72) ·
23 commits · base `staging` · all checks green · SonarCloud 0 issues

Three real bugs came out of it, each found by a test rather than by reading:
a 0 kg setoran that was being POSTed, a summary row that overflowed on every
real phone width, and a WhatsApp draft that could describe a setoran the server
never saw. The third is the one worth reading about — the code already carried
a comment claiming it was handled.

### The primitives bet, settled

Sprint 2's PIL-293 shipped three layout primitives with no call sites, on the
argument that PIL-340 would reuse them. That was a bet, and it mostly lost:

| Primitive | Outcome |
|---|---|
| `ContentBounds` | Adopted |
| `MasterDetailLayout` | Refuted — wrong breakpoint, wrong sizing model |
| `PageStateView` | Refuted — hides content a form must keep |

One for three. Both refuted widgets are still reasonable; they were just
designed against an imagined screen rather than a real one. The reasoning is
in [programming principles](../process/programming.md), where the claim had
been written down in advance precisely so it could be checked.

## A blocker that was not ours

Saving from a browser failed outright. The backend log showed four
`OPTIONS //api/v1/transaksi` preflights and no `POST` — the browser was
refusing to send the request, because it carries an `Idempotency-Key` header
and `CORS_ALLOW_ALL_ORIGINS=true` permits any origin but not arbitrary headers.

The useful part is why nobody had hit it: **CORS is browser-only.** The native
app sends no preflight, so the header always sailed through. Web is the first
client to send it, which is exactly the kind of thing a new platform surfaces.
Reported; the team was already handling it alongside a web deployment, and
asked for a local-only fix.

## Next

PIL-340 is in review. Native verification on a physical device is still
outstanding for both tickets — the APK builds and the phone paths are covered
by tests, but nothing has run on hardware.
