# Programming principles

The rubric wants a **named** principle, a **commit link**, and an explanation
of how it helps when the code changes. Naming a principle after the fact is
easy; the test is whether the principle actually drove the order the work was
done in.

## PIL-293 — what was applied, and where

### Single Responsibility — the destination list

`RoleNavigationBar` held two responsibilities: *which* destinations a role has,
and *how* they are drawn. Extracting `RoleDestinations` separated them.

**Commits:** [`c76eceb`](https://github.com/bank-sampah-PILAH/pilah-mobile/pull/70/commits)
(red) → [`a592653`](https://github.com/bank-sampah-PILAH/pilah-mobile/pull/70/commits) (green)

```dart
abstract final class RoleDestinations {
  static List<RoleDestination> forRole(String? role, {bool limitedNasabah = false}) =>
      role == 'nasabah'
          ? limitedNasabah ? _limitedNasabah : _nasabah
          : isStaff(role) ? _staff : const [];
}
```

**How it helps when things change.** Adding a destination for Pengurus is now a
one-line edit in one file, and all three navigation surfaces pick it up. Before
the split it would have meant editing each renderer and hoping they agreed.

### Open/Closed — two new renderers, zero edits to the old one

The rail was added **without modifying** `RoleNavigationBar`'s render path. Then
the drawer was added without modifying either.

**Commits:** [`f3b1242`](https://github.com/bank-sampah-PILAH/pilah-mobile/pull/70/commits)
(rail), [`d5123e8`](https://github.com/bank-sampah-PILAH/pilah-mobile/pull/70/commits) (drawer)

This is checkable rather than asserted: in the extraction commit the bar's
`build()` output is byte-for-byte unchanged, and the 18 existing navigation
tests passed with **none of them edited**. A principle that cannot be falsified
is not evidence; "the old tests still pass untouched" can be.

!!! important "The ordering is the actual evidence"
    I extracted the shared source *before* writing the rail, as its own
    red-green pair. If I had written the rail first and extracted afterwards, the
    diff would look similar but the claim would be hollow — the bar would have
    been modified and then tidied. Open/Closed is a statement about what you
    *did not have to touch*, which only the commit order can show.

### Strategy — one selector, three interchangeable renderers

`NavigationForm.resolve(isWeb:, width:)` returns which form applies;
`RoleNavigationBar`, `RoleNavigationRail` and `RoleNavigationDrawer` are the
interchangeable implementations, all reading the same destination source.

```dart
final form = NavigationForm.resolve(
  isWeb: isWebOverride ?? kIsWeb,
  width: MediaQuery.sizeOf(context).width,
);
```

The selection rule lives in one pure function with 11 unit tests, separate from
every widget. A fourth form would be a new case plus a new widget, with no
existing renderer touched.

### Dependency Inversion — the test seam

`MainPage` takes `@visibleForTesting bool? isWebOverride` instead of reading
`kIsWeb` directly deep in the tree. `kIsWeb` is a compile-time constant and
always `false` under `flutter test`, so without inverting that dependency the
entire web branch would be untestable.

This one has a cautionary tail: the *same* pattern applied to an
`@Injectable` constructor is what causes the
[DI regeneration defect](quality.md#ci-is-not-the-same-as-a-working-app)
elsewhere in this codebase. A seam on a widget parameter is fine; a seam on a
constructor that `injectable` reads is not. Same principle, and the context
decides whether it is correct.

### What I am not claiming

Flutter composition makes wrapper widgets like `ContentBounds` look like the
Decorator pattern, and `PageStateView` switching on an enum looks like State.
Both readings are a stretch — one is just how Flutter works, the other is a
`switch`. Claiming patterns that the framework hands you for free weakens the
claims that are real.

## PIL-340 — what I plan to claim

Stated in advance so it can be checked against the commits afterwards, rather
than reverse-engineered from them.

| Principle | Where it will apply |
|---|---|
| **SRP** | Split the setoran form's *layout* from its *validation*. A 487-line `StatefulWidget` currently holds the wide-screen arrangement, the item list state, and the validation rules together. The validation rules are the part worth isolating — they are testable without a widget. |
| **OCP** | Reuse `MasterDetailLayout` and `PageStateView` rather than branching inside the existing phone page. The phone path should end the ticket unmodified, same as the bottom bar did in PIL-293. |
| **Repository / Use Case layering** | `TransaksiCubit`, `AddTransaksiUseCase` and `TransaksiRepository` already exist. The web form consumes them unchanged — the evidence here is a *small* diff in the data layer, not a clever one. |
| **DIP** | The form depends on the cubit's state contract, not on a data source, so the wide-screen layout is testable with a mocked cubit. |

The honest risk: if the layout rework ends up needing changes inside
`TransaksiCubit`, the Open/Closed claim weakens and I should say so rather than
quietly rewording it. That is the point of writing the plan down first.

## PIL-340 — what actually happened

Written after slices 1–3, against the plan above. The point of stating claims
first is that they can come out wrong, and one did.

| Claim | Outcome |
|---|---|
| **SRP** | **Held.** [`SetoranDraft`](https://github.com/bank-sampah-PILAH/pilah-mobile/blob/9ce14d8/lib/features/transaksi/domain/entities/setoran_draft.dart) / `SetoranItemDraft` carry the rules, tested by [14 unit tests](https://github.com/bank-sampah-PILAH/pilah-mobile/blob/9ce14d8/test/features/transaksi/domain/setoran_draft_test.dart) with no widget built at all. It also caught a real bug: `num.tryParse('') ?? 0.0` means clearing the weight box yields 0, and nothing stopped that 0 kg setoran reaching the server. |
| **OCP** | **Held in substance, refuted in detail.** `TransaksiCubit`, `ItemSetoranCard` and `PilihNasabahSection` were never modified, and every pre-existing page and picker test passes unedited. But the specific reuse the plan named did not happen — see below. |
| **Repository / Use Case layering** | **Held.** `TransaksiCubit`, `AddTransaksiUseCase` and `TransaksiRepository` are untouched across all three slices. |
| **DIP** | **Not yet earned.** The wide-screen tests drive the real cubit over a stubbed API rather than a mocked cubit. True, but not the claim I made. |

### The claims that were wrong

PIL-293 deliberately shipped three layout primitives with no call sites, on the
argument that PIL-340 would adopt them. The plan named two of them. Both were
wrong.

**[`MasterDetailLayout`](https://github.com/bank-sampah-PILAH/pilah-mobile/blob/9ce14d8/lib/design/layout/master_detail_layout.dart#L36-L49)** does not fit, and could not:

- it splits at medium (600); the setoran form needs expanded (840). Between
  600 and 839, with the 256px rail, each column gets about 200px.
- its panes are proportional (3:2); a summary panel wants a fixed 340px. At
  1920 the detail pane would be 768px of mostly whitespace.
- the summary is an always-present aggregate, not the detail of a selected
  row — a different pattern wearing the same shape.

**[`PageStateView`](https://github.com/bank-sampah-PILAH/pilah-mobile/blob/9ce14d8/lib/design/layout/page_state_view.dart#L62-L84)** does not fit
either, and this one is worse, because using it would have made the screen
actively worse. It is built for a page loading data, and its own source says
so:

```dart
// Konten lama sengaja tidak ditampilkan di balik skeleton: angka basi
// yang terlihat seperti angka terkini lebih menyesatkan daripada menunggu.
PageState.loading => ListView.builder(... const SkeletonListItem()),
```

Hiding stale content is right for a list of figures and wrong for a form: it
would blank out what the pengelola just typed. `empty` has no meaning for a
save at all, and a failed save is already better served by a notification that
names the server's message while leaving the form intact to retry. Replacing
the form with a full-page error and a "Coba lagi" button would discard the
work.

So the scoreboard for the three primitives is **`ContentBounds` adopted, the
other two refuted** — one for three. Both remain reasonable widgets;
`MasterDetailLayout` is still right for a genuine list-and-detail screen
(PIL-337+), which is what PR #70 said it was for.

The honest conclusion is about the bet, not the widgets: **shipping a primitive
before its first caller means designing against an imagined screen.** Two of
the three guesses about that screen were wrong, and neither was discoverable
until a real one arrived.

!!! warning "A reviewer found this, not me"
    Heraldo flagged on PR #70 that a widget with no call sites is tested only
    against itself, so its tests keep passing after reality diverges. Checking
    that against PIL-340 is what exposed the stale claim — which had been
    published on this page for days while slice 2 had already hand-rolled a
    `Row` instead.

    The mechanism half-worked. Writing the plan down first is what made the
    claim *checkable*; it did not make me check it. That is a weaker result
    than the one I wanted, and pretending otherwise would be the exact failure
    this page exists to catch.

### What appeared that was not planned

**Strategy, again.** [`PickerPresentation.resolve(width:)`](https://github.com/bank-sampah-PILAH/pilah-mobile/blob/9ce14d8/lib/design/layout/picker_presentation.dart#L53-L57) picks the container
for a modal picker — bottom sheet on a phone, centred dialog in a browser —
with `showAdaptivePicker` owning all the container chrome. Same shape as
`NavigationForm` in PIL-293, and arrived at for the same reason: the rule is
worth deciding in one pure function instead of a `MediaQuery` check at each of
the four picker call sites.

Worth noting the discriminator differs on purpose. Navigation is decided by
**platform** (the lead dev's rule — bottom bar in the app, left-hand in a
browser), because it stays on screen. A picker is transient, so Material ties
it to **width**. The two agree wherever it matters, since a native phone is
always compact.

The named risk — needing changes inside `TransaksiCubit` — did not materialise.
