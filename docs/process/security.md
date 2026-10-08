# Security

The rubric asks for the OWASP Top 10 item to be named **in the commit message
body**, not only in the PR description. Two of my tickets have a genuine
security argument; the rest do not, and I would rather say so than manufacture
one.

## What the rubric actually asks for

Worth stating plainly, because I had this wrong. The Part B security criterion
is scored by **how many** of the OWASP Top 10 the code demonstrably prevents,
named in the commit message:

| Level | Requirement |
|---|---|
| 1 | Code preventing **1** of the Top 10, named in the commit message |
| 2 | Code preventing **at least 5** of the 10, named in the commit messages |
| 3 | A scan with Metasploit or similar, explained, with a patch plan |
| 4 | Penetration testing with video evidence that the system is secure |

PIL-293 names **A01 only**, so on its own it sits at level 1. Reaching level 2
needs five, and levels 3 and 4 need a scanning tool that nothing in this
project currently uses — no amount of careful commit writing substitutes for
it.

Note the wording: *"has shown some/all of the code that **already** prevents"*.
Pointing at existing protections counts, not only at code written this week. So
below I mark what I wrote against what I am citing, because those are different
claims.

## A01 — Broken Access Control

### PIL-293: one source of truth for role-to-destination mapping

Adding a second navigation surface to an app is an access-control risk in a
quiet way. If the bottom bar and the rail each held their own list of
destinations, the two could drift — and the drift would be a role seeing a
destination it should not reach.

The mitigation is structural rather than a check: *which* destinations a role
has is decided in exactly one place.

```dart
static List<RoleDestination> forRole(String? role, {bool limitedNasabah = false}) =>
    role == 'nasabah'
        ? limitedNasabah ? _limitedNasabah : _nasabah
        : isStaff(role)
            ? _staff
            : const [];          // <- fails closed
```

Three properties, each pinned by a test:

- **Fails closed.** An unrecognised role gets `const []`, not the staff menu.
  Test: *"an unknown role gets nothing rather than a default menu"*.
- **Least privilege for unapproved members.** A nasabah whose membership is not
  yet approved gets only Beranda and Profil.
- **No session, no navigation.** The rail renders `SizedBox.shrink()` for a role
  without navigation and when there is no session.

!!! important "This is defense in depth, not the control"
    A UI that hides a menu item is not access control — the real control is
    server-side authorization plus the router redirect. Claiming otherwise would
    be overclaiming, and the commit bodies say so explicitly: *"Kontrol akses
    sebenarnya tetap di server dan redirect router; ini lapisan UI."*

The OWASP line appears in four commits: both `RoleDestinations` commits and both
rail commits. The other eighteen — layout primitives, breakpoints, content
bounds — have no security angle, so they do not claim one.

### PIL-223 / PIL-288: who may edit a nasabah profile

Once a nasabah has their own account, the pengurus loses the right to edit their
personal data. That is access control in the ordinary sense: enforced
server-side with a 403, and reflected in the client as a read-only form
(PIL-206).

The detail that matters: the 403 fires only when a locked value **differs** from
what is stored. Submitting an unchanged locked field is not an attempt to change
it.

## Secrets handling

Not an OWASP item as such, but it is where mistakes are cheapest to make:

- `.env`, Firebase credentials and `google-services.json` are never committed.
  `.env` is a declared Flutter asset, so builds fail without it — which makes
  the temptation to commit it real.
- CI materialises `google-services.json` from a repository secret. When I could
  not build a debug APK locally without it, I reported the blocker rather than
  working around it.
- Local QA runs with a permissive CORS setting and a fake-Google-token flag.
  Both are development-only and documented as such, not as defaults.

## The five for PIL-340

PIL-340 (setoran recording on web) is where a level-2 claim becomes honest,
because the feature touches money. Planned, with my confidence in each:

| OWASP | What prevents it | Strength |
|---|---|---|
| **A01** Broken Access Control | `TransaksiBaruPage` is in the router's `staffPaths` set, so a nasabah is redirected before it loads; destinations come from `RoleDestinations`, which fails closed | strong — *mine* |
| **A04** Insecure Design | The server fills each item's price from the master *jenis sampah* record. The client's form total is display only and is never trusted as the amount | strong — *cited*, from PIL-168 |
| **A08** Data Integrity Failures | The success screen shows the server's returned `totalNilai` and `saldoSetelah`, not the form's own arithmetic, so a client-side rounding slip cannot misreport a balance | strong — *mine* |
| **A05** Security Misconfiguration | Demo login is compiled out of release builds (`supportsDemoLogin => !kReleaseMode && ...`), so test accounts cannot exist in production | real — *cited* |
| **A07** Authentication Failures | A web session whose role is not web-supported is logged out with a warning rather than shown a broken page | real — *cited* |

**What I am not claiming.** A03 (Injection) has no honest frontend story here:
Flutter renders text as text with no HTML sink, and the client sends typed JSON
through Dio rather than building queries. The real injection surface is
server-side and is not mine this ticket. A02, A06, A09 and A10 have nothing to
do with this feature. Five is what the evidence supports; padding to ten would
be the kind of claim a grader is right to knock down.

## Where I slipped

!!! warning "PIL-168 — OWASP in the PR, not the commit"
    The A01 note appeared only in the pull request body. The rubric asks for the
    commit message.

!!! warning "PIL-293 — missed it again, then fixed it"
    All 22 commits shipped with no OWASP line. It was caught after the PR was
    open and green, and fixed by rewriting the four relevant commit messages.
    The habit I have taken from it: decide the OWASP item while *planning the
    slices*, by asking of each one — does this touch who-can-reach-what, what
    gets stored, or what gets rendered from input?
