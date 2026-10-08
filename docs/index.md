# PILAH 2.0 — engineering documentation

**Melanton Gabriel Siregar** · Kelompok 2 · Proyek Perangkat Lunak, Fakultas Ilmu Komputer UI

PILAH 2.0 is a *bank sampah* (waste bank) management system: members deposit
sorted waste, staff record it by weight and price, and balances accumulate
until the member withdraws. I work across both repositories — the Django
backend and the Flutter client.

This site is my own record of that work: what I shipped, how I shipped it, and
the evidence behind each claim. It is written to be checked, so wherever a
claim can be traced to a commit, a PR or a test, the link is there.

!!! note "What this site is not"
    It is not the team's documentation. Architecture decisions, the PRD and the
    SDS live with the team. This is the individual-report side: my
    contributions and my process.

## At a glance

| | |
|---|---|
| **Role** | Fullstack — Django REST backend and Flutter client |
| **Repositories** | `pilah-be` (Django), `pilah-mobile` (Flutter, mobile + web) |
| **Merged PRs** | 5 backend, 3 mobile |
| **Open** | 1 mobile (PIL-293) |
| **Tickets delivered** | PIL-168, 224, 223, 281, 288 (backend) · PIL-214, 206, 288 (mobile) |
| **In progress** | PIL-293 — responsive web shell for Pengurus |

## Sprint schedule

| Sprint | Dates (2026) | My focus |
|---|---|---|
| Sprint 1 | 15 Sep – 1 Oct | Money rules, deposit validation, nasabah profile permissions |
| **Sprint 2** | **6 – 22 Oct** | **Flutter web dashboard for Pengurus** |
| Sprint 3 | 27 Oct – 12 Nov | — |
| Sprint 4 | 17 Nov – 3 Dec | Hardening only, no new features |

Final presentation: 10 December 2026.

## The through-line

Three themes connect most of what I have built.

**Money must be exact.** Rupiah is a whole number in this system — no sen, no
half-rupiah — and calculations round *down*, never to nearest. I implemented
that rule in PIL-168 and it has since become shared infrastructure:
`shared_kernel/kalkulasi.bulatkan_rupiah` is now the single place the rule
lives, and other people's code is being refactored onto it. See
[Sprint 1](contributions/sprint-1.md).

**Permissions belong where they can be enforced.** PIL-223 and PIL-288 are both
about *who may change what*, and both resolve to the same shape: the server
decides, the client reflects. When I later built role-aware navigation for the
web shell, the same instinct produced a single source of truth for
role-to-destination mapping. See [Security](process/security.md).

**Tests come first, and the commits show it.** Every delivery here is a
sequence of `red` → `green` → `refactor` commits where the failing test is a
separate commit from the code that makes it pass. See [TDD](process/tdd.md).

## Finding your way around

<div class="grid cards" markdown>

- :material-source-branch: **[Contributions](contributions/index.md)**
  Every ticket, every PR, what each one actually changed.

- :material-flask: **[How I work](process/index.md)**
  TDD, security, review and quality — with real examples, including the
  mistakes.

- :material-clipboard-check: **[IR evidence](ir/index.md)**
  Mapped to the Part B and Part C criteria, with links.

</div>
