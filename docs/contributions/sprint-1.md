# Sprint 1 — 15 Sep to 1 Oct 2026

Five backend tickets and three mobile ones. Two threads run through them:
getting money arithmetic right, and getting profile permissions right.

## PIL-168 — use master price and round rupiah down

[`pilah-be` #22](https://github.com/bank-sampah-PILAH/pilah-be/pull/22)

The deposit calculation trusted the price the client sent. That is both a
correctness problem and a trust problem: a client could post any price it
liked. This ticket made the **bank's own master price** the only source, and
fixed the rounding direction.

The rounding question was genuinely contested. PIL-138's notes implied
`ROUND_HALF_UP`. Tristan, as lead dev, settled it on 17 Sep: rupiah is a whole
number — smallest unit Rp 1, no sen — and price calculations always round
**down**. Rounding half-up would have credited members fractions of a rupiah
they had not earned, and on a system migrating legacy PILAH 1.0 balances that
carry sen, those fractions accumulate.

What came out of it:

```python
RUPIAH = Decimal(1)

def bulatkan_rupiah(nilai: Decimal) -> Decimal:
    """Bulatkan `nilai` ke bawah menjadi rupiah penuh."""
    return nilai.quantize(RUPIAH, rounding=ROUND_DOWN)
```

That helper now lives in `shared_kernel/kalkulasi.py` and is the single place
the rupiah rule exists. Other people's code is still being migrated onto it —
Vegard's [#89](https://github.com/bank-sampah-PILAH/pilah-be/pull/89) moves the
pencairan paths across, which is the PR I reviewed in
[Peer review](../process/peer-review.md).

!!! info "Why `ROUND_DOWN` and not `ROUND_FLOOR`"
    They differ for negative values: `ROUND_DOWN` truncates toward zero,
    `ROUND_FLOOR` toward negative infinity. Balances are guarded against going
    negative before rounding, so either would work today — but `ROUND_DOWN`
    states the intent ("drop the fraction") rather than a direction on the
    number line.

## PIL-224 — validate deposit weight limits and usable price

[`pilah-be` #24](https://github.com/bank-sampah-PILAH/pilah-be/pull/24), stacked on #22

Boundary work. A deposit needs a weight within sane limits and a price that is
actually usable, and the interesting cases are all at the edges: zero weight,
negative weight, a weight just inside and just outside the limit, a jenis
sampah with no price set. This is where I started writing on-point and
off-point tests deliberately rather than by accident — see
[TDD](../process/tdd.md).

Stacked on #22 because it needs the price source from it. First time I used a
stacked PR on this project.

## PIL-223 — restrict profile edits for nasabah with an account

[`pilah-be` #35](https://github.com/bank-sampah-PILAH/pilah-be/pull/35)

Once a nasabah has their own account, the pengurus should no longer edit their
personal data — the member owns it. The pengurus keeps membership data:
member number, active status, membership status.

Locked once an account exists: `nama`, `jenis_kelamin`, `tanggal_lahir`,
`alamat`, `no_hp`. Still editable: the membership fields.

Two decisions worth recording, both made with Pascal:

**Rejection shape.** One 403 for the whole request, matching the existing
"Nasabah nonaktif tidak bisa diedit" behaviour, and only when a locked value
actually *differs* from what is stored. Sending an unchanged locked field is
not an attempt to change it, so it should not fail.

**`email` was deferred.** It could not be locked yet because PIL-154 had not
merged. That PR adds `Nasabah.email` as a globally unique field and links the
account at first Google login via `email__iexact` — so changing it on a nasabah
who already has an account would break or mis-point that link. For a nasabah
*without* an account it must stay editable, because that is how the pengurus
invites them. I followed it up rather than guessing, and it landed in PIL-288.

Field split backed by PRD F21/F25.

## PIL-281 — expose nasabah account-linkage on the API

[`pilah-be` #58](https://github.com/bank-sampah-PILAH/pilah-be/pull/58)

The client could not tell whether a nasabah had an account, so it could not
know which fields to disable. This adds `punya_akun` to the list and detail
responses — the API telling the client what the server will enforce, instead of
the client guessing.

## PIL-288 — profile editing, email stays locked

[`pilah-be` #65](https://github.com/bank-sampah-PILAH/pilah-be/pull/65) ·
[`pilah-mobile` #49](https://github.com/bank-sampah-PILAH/pilah-mobile/pull/49)

The follow-up I promised in PIL-223. PIL-154 had merged, so `email` could now
be locked for the right reason, and the rest of profile editing was restored.
Backend and client in the same ticket: the server enforces, the form reflects.

## PIL-214 — page the nasabah list for pengurus

[`pilah-mobile` #32](https://github.com/bank-sampah-PILAH/pilah-mobile/pull/32)

The nasabah list loaded every row at once. This paged it. The problems worth
solving were not the happy path but the state ones: keeping the current page
when a filter changes, discarding a response that arrives after the user has
moved on, and not duplicating rows when a page is appended twice.

Those three are exactly what I checked for when reviewing Rifqi's paging work
in [mobile #69](https://github.com/bank-sampah-PILAH/pilah-mobile/pull/69).

!!! warning "A correction I had to take"
    On this ticket the user asked whether I actually understood red-green-refactor,
    because I had been committing in a way that did not show the cycle. That was
    fair. It is why the commit discipline described in [TDD](../process/tdd.md)
    is explicit now rather than assumed.

## PIL-206 — read-only nasabah profile for accounts the pengurus does not own

[`pilah-mobile` #38](https://github.com/bank-sampah-PILAH/pilah-mobile/pull/38)

The client side of the PIL-223 rule: when `punya_akun` is true, the profile
form renders read-only instead of letting the pengurus type into fields the
server will reject. Same rule, two layers — the server refuses, and the UI does
not offer.
