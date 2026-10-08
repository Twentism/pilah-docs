# Code quality

## Gates I run before pushing

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

`makemigrations --check` is in there because a model change without a migration
passes every other check and then breaks the next person to pull.

## SonarCloud

Organisation `bank-sampah-pilah`, projects `bank-sampah-PILAH_pilah-be` and
`bank-sampah-PILAH_pilah-mobile`. It has native branch analysis, so a clean
report on a feature branch is directly checkable before review:

```
https://sonarcloud.io/project/issues?id=<project-key>&resolved=false&inNewCodePeriod=true&branch=<branch>
```

PIL-293 (PR #70) passes.

!!! note "This moved mid-project"
    Analysis used to be a self-hosted SonarQube with per-branch project keys.
    It migrated to SonarCloud in October. Two things got better: branch analysis
    is native, and the Issues page author facet works — so per-developer
    evidence no longer needs the inverse-query workaround Sprint 1 used.

## Coverage

The team standard is 100% on new code. The enforced CI gates are lower —
mobile has a 25% floor plus `diff-cover`, backend 80% plus `diff-cover` — so
100% is a visible norm rather than a hard gate, and dropping below it is
noticeable.

Backend reached 100% overall; mobile reached it in a dedicated PR.

## Reading a failing suite honestly

PIL-293 taught me to be careful about the phrase "pre-existing failure".

Locally the suite showed 9 failures; in CI it showed 2. Neither number was
wrong — they measure different things:

- **7 were environmental.** Onboarding data-layer tests that depend on my local
  `.env`. CI uses `.env.example` and never sees them.
- **2 were mine.** A real regression, in files my local verification had not
  covered. I had called all nine pre-existing. CI was right and I was wrong.

The useful lesson is about the claim, not the tests: *"I verified it"* has to
name what was verified. A check across three files cannot speak for a suite.

## CI is not the same as a working app

PIL-293 surfaced a defect that every gate above would miss. `AuthenticationBloc`
has an optional named parameter used as a test seam; `injectable` reads it and
regenerates the DI config as `isWebOverride: gh<bool>()`, which kills the app at
startup.

CI runs `build_runner`, so it *does* build on the broken config — and analyze
plus 1338 tests pass, because CI never runs the app. GetIt resolution is never
exercised.

Worth holding on to: a green pipeline means the checks passed, not that the
application starts.
