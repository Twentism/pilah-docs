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

    [`navigation_form.dart`](https://github.com/bank-sampah-PILAH/pilah-mobile/blob/9d8c439/lib/design/layout/navigation_form.dart#L26-L35).
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

    [`role_destinations.dart`](https://github.com/bank-sampah-PILAH/pilah-mobile/blob/9d8c439/lib/features/main/presentation/widgets/role_destinations.dart#L36-L46).
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

### The same three, from PIL-340

PIL-293's examples are about layout. These are about a rule that moves money,
so the stakes read differently. All three come from
[`setoran_draft_test.dart`](https://github.com/bank-sampah-PILAH/pilah-mobile/blob/9ce14d8/test/features/transaksi/domain/setoran_draft_test.dart),
which builds no widget at all — [positive](https://github.com/bank-sampah-PILAH/pilah-mobile/blob/9ce14d8/test/features/transaksi/domain/setoran_draft_test.dart#L26-L31), [negative](https://github.com/bank-sampah-PILAH/pilah-mobile/blob/9ce14d8/test/features/transaksi/domain/setoran_draft_test.dart#L67-L73), [corner](https://github.com/bank-sampah-PILAH/pilah-mobile/blob/9ce14d8/test/features/transaksi/domain/setoran_draft_test.dart#L101-L113).

=== "Positive"

    ```dart
    test('a nasabah and one weighed item is submittable', () {
      final subject = draft();
      expect(subject.problems, isEmpty);
      expect(subject.isValid, isTrue);
      expect(subject.invalidItemIndexes, isEmpty);
    });
    ```

    The ordinary case: someone chosen, something weighed. It asserts on all
    three accessors rather than just `isValid`, because the form needs
    `invalidItemIndexes` to decide which card to redden — a happy path that
    only checked the boolean would let that drift.

=== "Negative"

    ```dart
    test('an item left at zero weight', () {
      final subject = draft(
          items: const [SetoranItemDraft(jenisSampahId: jenis, berat: 0)]);
      expect(subject.problems, contains(SetoranProblem.nonPositiveBerat),
          reason: 'clearing the weight box yields 0 and must not be submitted');
      expect(subject.invalidItemIndexes, {0});
    });
    ```

    This is the test that found a real bug. `num.tryParse('') ?? 0.0` means an
    emptied weight box is `0`, and the old guard never looked. The negative
    case here is not a hypothetical bad input — it is what happens when a
    pengelola selects the weight and presses backspace.

=== "Corner"

    ```dart
    test('zero is rejected and the smallest step above it is accepted', () {
      expect(
        draft(items: const [SetoranItemDraft(jenisSampahId: jenis, berat: 0)])
            .isValid,
        isFalse,
      );
      expect(
        draft(items: const [SetoranItemDraft(jenisSampahId: jenis, berat: 0.1)])
            .isValid,
        isTrue,
        reason: 'the rule is "more than zero", not "at least one kilo"',
      );
    });
    ```

    Both halves are needed. Without the second, `berat >= 1` would also pass —
    and bank sampah weigh in hundreds of grams, so that reading of the rule
    would quietly reject real setoran. The corner case is the pair, not either
    value alone.

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

## Test doubles: what is faked, and what it costs

The rubric asks which tests use mocks or stubs. The more useful answer is
*which layer* each one fakes, because that decides what the test can still
catch.

!!! abstract "The difference, in plain terms"
    A **stub** is a stand-in that *answers*. You wire it in, it hands back a
    prepared reply, and you judge the result by looking at what the app did
    next. A pretend shopkeeper who always sells you the same loaf.

    A **mock** is a stand-in that *remembers*. You wire it in, let the app use
    it, and then ask it what it was told to do. A pretend shopkeeper who writes
    down everything you ordered so you can check the list afterwards.

    Same idea — something fake standing where the real thing goes. The
    difference is where you look for the answer: at the app, or at the fake.

### Level 0 — no double at all

[`setoran_draft_test.dart`](https://github.com/bank-sampah-PILAH/pilah-mobile/blob/9ce14d8/test/features/transaksi/domain/setoran_draft_test.dart),
14 tests.

**Practically:** nothing is faked. The test hands the rule a nasabah and some
weights and checks the verdict, the way you would check a calculator.

[`SetoranDraft`](https://github.com/bank-sampah-PILAH/pilah-mobile/blob/9ce14d8/lib/features/transaksi/domain/entities/setoran_draft.dart)
has no collaborators: it takes values and answers questions about them.
Nothing to fake, so nothing is faked.

**Implication.** Fast, and they cannot rot — there is no seam to drift. They
also prove nothing about wiring: a perfectly correct `SetoranDraft` that no
page ever calls would pass all 14. That is exactly why slice 1 was followed by
a page test asserting no request was sent.

### Level 1 — a stub at the transport boundary

[`stub_api.dart:53`](https://github.com/bank-sampah-PILAH/pilah-mobile/blob/9ce14d8/test/support/stub_api.dart#L53-L85):

```dart
class StubApi implements HttpClientAdapter {
  void on(String method, String path, {int status = 200, Object? json}) {
    _routes['${method.toUpperCase()} $path'] = (_) => ResponseBody.fromString(
          jsonEncode(json), status, ...);
  }
}
```

**Practically: a pretend server.** The app genuinely builds its request, fills
in the headers, and sends it. The request simply never leaves the machine — a
canned reply we wrote is handed back instead. Everything the app does on the
way out and on the way back is the real code.

This is a **stub**, not a mock: it answers, and it keeps a note of what it was
asked, but no test ever interrogates *it*. It replaces Dio's
`HttpClientAdapter`, the lowest layer in the app — the part that would
otherwise open a socket.

Everything above it is real.
[`transaksi_support.dart:13`](https://github.com/bank-sampah-PILAH/pilah-mobile/blob/9ce14d8/test/support/transaksi_support.dart#L13-L23):

```dart
TransaksiCubit buildTransaksiCubit(StubApi api) {
  final repository =
      TransaksiRepositoryImpl(TransaksiRemoteDataSourceImpl(api.network));
  return TransaksiCubit(
    GetTransaksiUseCase(repository),
    GetTransaksiDetailUseCase(repository),
    AddTransaksiUseCase(repository),
    ...
  );
}
```

So a page test exercises the genuine cubit, use case, repository, data source
and JSON serialisation. Only the socket is fake.

**Implication, and the reason this is the default here.** Assertions are about
*state and traffic*, not about calls:

```dart
expect(api.requests.where((r) => r.method == 'POST'), isEmpty,
    reason: 'a 0 kg setoran must not be sent at all');
```

That sentence is the actual requirement. Had the page test mocked
`TransaksiCubit` instead, the strongest available assertion would have been
"`addTransaksi` was not called" — true, but one layer away from what matters,
and still green if a request reached the network by some other path. The stub
lets the test assert at the boundary the requirement is written about.

The cost is real: these tests are slower, and they can fail for reasons outside
the widget under test. On this project they have repeatedly earned it.

### Level 2 — a mock, for a side effect that leaves the app

[`transaksi_baru_page_test.dart:33`](https://github.com/bank-sampah-PILAH/pilah-mobile/blob/9ce14d8/test/features/transaksi/presentation/pages/transaksi_baru_page_test.dart#L33-L35):

```dart
class _MockUrlLauncher extends Mock
    with MockPlatformInterfaceMixin
    implements UrlLauncherPlatform {}
```

**Practically: a stand-in that keeps the receipt.** A test cannot really open
WhatsApp, so we put a fake in its place, let the app press the button, and
afterwards ask the fake: *were you told to open a link, and which one?*

Installed at
[`:63`](https://github.com/bank-sampah-PILAH/pilah-mobile/blob/9ce14d8/test/features/transaksi/presentation/pages/transaksi_baru_page_test.dart#L63)
and interrogated at
[`:441`](https://github.com/bank-sampah-PILAH/pilah-mobile/blob/9ce14d8/test/features/transaksi/presentation/pages/transaksi_baru_page_test.dart#L441-L443)
and
[`:506`](https://github.com/bank-sampah-PILAH/pilah-mobile/blob/9ce14d8/test/features/transaksi/presentation/pages/transaksi_baru_page_test.dart#L506-L508):

```dart
final launched = verify(() => launcher.launchUrl(captureAny(), any()))
    .captured
    .single as String;
expect(launched, contains('Botol'));
```

Opening WhatsApp hands control to another application. There is no resulting
state inside our app to inspect, so the only observable fact is *that the call
happened, with this URL*. That is behaviour verification, and a mock is the
right tool for it.

**Implication.** This test is coupled to the shape of the call. Change
`launchUrl`'s signature, or route the deeplink through a wrapper, and it breaks
even though the behaviour is identical. That is the price of asking the fake
instead of looking at the result, and it is worth paying only where there is no
result to look at.

### Level 3 — a mock to reach a state the stub cannot produce

[`pilih_nasabah_flow_test.dart:15`](https://github.com/bank-sampah-PILAH/pilah-mobile/blob/9ce14d8/test/features/transaksi/presentation/widgets/pilih_nasabah_flow_test.dart#L15-L16),
used at
[`:121`](https://github.com/bank-sampah-PILAH/pilah-mobile/blob/9ce14d8/test/features/transaksi/presentation/widgets/pilih_nasabah_flow_test.dart#L120-L129):

```dart
final broken = _MockNasabahCubit();
whenListen(broken, const Stream<NasabahState>.empty(),
    initialState: NasabahInitial());
when(() => broken.loadActiveNasabah())
    .thenAnswer((_) async => throw StateError('rusak'));
```

**Practically: a stand-in told to break on purpose.** We want to see what the
screen shows when something fails in a way the network never fails, so we swap
in a component instructed to throw.

The pretend server can return a 403, a 500, or malformed JSON — all of which
arrive as `NetworkException`. It cannot produce a bare `StateError`, because
that is not a thing a transport returns. The picker has a branch for exactly
that case:

```dart
error is NetworkException ? error.displayMessage : error.toString()
```

Without a mock, that second path is unreachable and untested.

**Implication.** The mock exists to reach one otherwise-dead line, and the test
still asserts on rendered text rather than on calls. It is the narrowest use I
reached for.

### The rule I follow

1. **No double** when the unit has no collaborators.
2. **Stub** by default, as low as possible — fake the socket, keep the app.
3. **Mock** only when there is no observable result: a side effect that leaves
   the app, or a state the stub physically cannot produce.

The failure mode this avoids is a suite that mocks the layer directly beneath
the one under test. It passes forever, breaks on every refactor, and never
catches an integration bug — the three worst properties a test can have.

!!! note "Both kinds, in one test"
    The WhatsApp-snapshot test needs a request held open mid-flight, and needs
    to see which URL was launched. The stub supplies the first
    (`api.latency = const Duration(seconds: 1)` — the pretend server answering
    slowly on purpose); the mock supplies the second. They are not
    alternatives, they answer different questions.

### A double that made a test lie

`StubApi` is not the only fake in play. The WhatsApp template comes from
[`profile_support.dart:37`](https://github.com/bank-sampah-PILAH/pilah-mobile/blob/9ce14d8/test/support/profile_support.dart#L34-L40),
whose default is:

```dart
'template': 'Halo {nama}'
```

My first version of the snapshot test asserted the launched URL contained
`'Botol'`. It never could: that template has no item placeholder, and `{nama}`
is not even a key the code substitutes. The test failed before the fix and
would have failed after it — it was not measuring the behaviour at all.

Fixed by having the test set the fixture it actually depends on:

```dart
stubProfile(api, wa: {'template': 'Setoran: {daftar_item}', ...});
```

The lesson is specific to doubles: **a test is only as meaningful as the
fixture it runs against**, and a shared default fixture is a comfortable place
for a test to quietly stop testing anything.

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
