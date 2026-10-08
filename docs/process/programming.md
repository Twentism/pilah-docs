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
