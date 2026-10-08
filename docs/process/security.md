# Security

The rubric asks for the OWASP Top 10 item to be named **in the commit message
body**, not only in the PR description. Two of my tickets have a genuine
security argument; the rest do not, and I would rather say so than manufacture
one.

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
