# Test-driven development

## The commit convention

The cycle is visible in the git history, because that is the only way a claim
to practise TDD can be checked:

| Prefix | Means |
|---|---|
| `red(scope)` | A failing test for **one** behaviour. Run, and seen to fail. |
| `green(scope)` | The smallest change that makes it pass. |
| `refactor(scope)` | Real cleanup, no behaviour change. |

Two rules I hold to:

- **The test and the implementation are never in the same commit.** If they
  are, nobody can tell whether the test was written first.
- **One behaviour per cycle.** Bundling slices makes the history unreadable
  and hides which test drove which line.

## What it looks like in practice

PIL-293's 22 commits, abbreviated:

```
red(layout)    klasifikasi lebar layar belum ada
green(layout)  namai tiga kelas lebar layar dalam satu enum
red(layout)    lebar belum terbaca dari widget tree
green(layout)  baca breakpoint lewat MediaQuery milik element
red(layout)    lebar konten belum dibatasi di layar lebar
green(layout)  batasi dan pusatkan konten pada blok expanded
red(nav)       daftar destinasi belum punya sumber bersama
green(nav)     pindahkan daftar destinasi ke sumber bersama
red(nav)       belum ada bentuk navigasi untuk jendela lebar
green(nav)     tambahkan navigation rail untuk jendela lebar
red(shell)     bentuk navigasi belum mengikuti lebar jendela
refactor(nav)  sematkan viewport telepon pada mount test navigasi
green(shell)   pilih bentuk navigasi dari lebar jendela
...
```

Each `green` commit body records the commands actually run and their output,
so the claim is checkable:

```
flutter test test/design/layout/layout_breakpoint_test.dart -> 10 lulus
flutter test test/features/main/role_navigation_test.dart   -> 15 lulus, tidak ada regresi
flutter analyze lib/design/layout test/design/layout        -> No issues found
```

## Positive, negative, corner — one each

The rubric asks for all three per feature. Taking PIL-293's navigation form as
the feature, here is exactly one of each, so the distinction is concrete rather
than asserted.

=== "Positive"

    The expected path, with valid input.

    ```dart
    test('a desktop window gets the labelled rail', () {
      expect(
        NavigationForm.resolve(isWeb: true, width: 1440),
        NavigationForm.railExtended,
      );
    });
    ```

    A browser at a normal desktop size gets the labelled rail. If only this
    kind of test existed, the feature would look finished and be wrong in
    three other places.

=== "Negative"

    Invalid or unrecognised input, where the right answer is a refusal.

    ```dart
    test('an unknown role gets nothing rather than a default menu', () {
      expect(RoleDestinations.forRole('satpam'), isEmpty);
    });
    ```

    An unrecognised role gets an empty list, **not** the staff menu. The
    negative case is the one with security weight: the dangerous failure here
    is not a crash, it is a helpful default.

=== "Corner"

    Valid input sitting exactly on a decision boundary.

    ```dart
    test('the drawer-to-rail boundary belongs to the rail', () {
      expect(NavigationForm.resolve(isWeb: true, width: 599),
          NavigationForm.drawer);
      expect(NavigationForm.resolve(isWeb: true, width: 600),
          NavigationForm.railCollapsed);
    });
    ```

    599 and 600 are both ordinary widths; what makes this a corner case is that
    the answer changes between them. A test at 400 and 1400 passes whether the
    comparison is `>` or `>=`. This one does not.

### How I find the corner cases

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

On PIL-293 that third question is what surfaced the degenerate-width case, and
question 1 is what produced the on-point/off-point table below.

## Boundary testing, not sample testing

Where an input is continuous, I partition it and test the boundaries rather
than picking values that feel representative. PIL-293's window width:

| Partition | On-point | Off-point |
|---|---|---|
| compact | 599 | 600 |
| medium | 600, 839 | 599, 840 |
| expanded | 840 | 839 |

PIL-224 (deposit weight limits) is the backend version of the same habit: zero
weight, negative weight, just inside the limit, just outside, and a jenis
sampah with no usable price.

## Proving a test actually guards the line

A test that passes is not evidence that it would fail. When I add a test to
pin existing behaviour, I break the line on purpose and watch it fail, then
restore it. If it does not fail, it was not guarding anything.

I look for the same thing when reviewing. Vegard did exactly this on
[be #89](https://github.com/bank-sampah-PILAH/pilah-be/pull/89) — removed the
rounding, saw `315600.75 != 315600.00`, restored it — and it is why his
"no behaviour change" claim was believable rather than asserted.

## Where I got this wrong

!!! warning "PIL-214 — the correction"
    I was committing in a way that did not show the red-green-refactor cycle,
    and was asked directly whether I understood it. That was a fair question.
    The convention above is written down *because* of that, rather than being
    something I claim to have always done.

!!! warning "PIL-293 — an amend that mixed a cycle"
    A `git commit --amend` pulled a formatting reflow into a `green` commit,
    which put test and implementation changes in the same commit. I rebuilt
    both commits cleanly (`git reset --soft HEAD~2`) rather than leave it.

!!! warning "PIL-293 — a wrong claim about pre-existing failures"
    I told the user all nine failing tests were pre-existing and said I had
    verified it. Seven were; two were my own regression, in files my
    verification had not covered. CI caught it. The lesson is narrow and
    useful: "I verified it" has to name *what* was verified, because a check
    over three files cannot speak for a suite.
