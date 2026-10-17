# Address equality — every `Address`-vs-`Address` comparison in the escrow contract

Issue #122 asked whether any identity check in `contracts/escrow/src/lib.rs`
compares `Address` values in a way that is not real Soroban address equality —
the classic failure being a comparison that goes through a text form
(`strkey`/`to_string`/`format!`), where two spellings of one account, or one
truncated spelling of two accounts, can decide who is a party.

This is the audit note. The tests that make it re-runnable live in
[`contracts/escrow/tests/address_equality.rs`](../contracts/escrow/tests/address_equality.rs),
and CI runs them as their own job (`#122 Address equality audit` in
[`.github/workflows/test.yml`](../.github/workflows/test.yml)).

**Finding: no such comparison exists, so there is nothing to fix.** Every
address identity decision in the contract is either delegated to the host
(`require_auth`) or performed with `soroban_sdk::Address`'s `PartialEq`, which
compares the underlying `ScAddress` value. `src/lib.rs` is `#![no_std]` and has
no `String` in scope; it never calls `to_string`, `to_strkey`, `format!`,
`from_str`, or any byte-level conversion on an `Address`.

## The audit

Enumerated from the current `src/lib.rs`, one row per place two addresses — or
an address and a stored address — could be judged equal:

| Site | What it compares | Verdict |
| --- | --- | --- |
| `initialize`, `lock`, `claim_tranche` (via `claim_advance_1`/`_2`), `reclaim_tranche`, `fund_remainder`, `reclaim_on_nondelivery`, `mark_checkpoint`, `ready_for_delivery`, `confirm_delivery`, `set_allocation`, `fund_fx_shortfall` | no comparison at all: `Address::require_auth()` hands the address to the host's auth layer, which matches the authorizing principal against the address value in the auth entry | type-safe, host-enforced |
| `flag_dispute`'s `NotAParty` gate | `flagger != c.buyer && flagger != c.cooperative && flagger != c.warehouse_operator` | `Address: PartialEq` → host equality over the `ScAddress` value. Not string- or byte-based |
| `require_once_each` (backs `cancel`, `reassign_buyer`, `resolve_dispute`) | `addresses[..i].iter().all(\|seen\| seen != addr)` | same `PartialEq`, through one layer of `&`; no coercion |
| `settle`, `resolve_fx_shortfall`, all token transfers | addresses used only as keys/arguments (`token::Client::new(&env, &c.token)`, transfer `from`/`to`) | no equality comparison |
| `get_commitment`, storage reads | `Address` fields round-tripped through `contracttype` as `ScVal::Address` | value preserved across the round trip (pinned by a test) |

`BytesN<32>` appears exactly once — `AllocationMember::member_hash` — and is
compared nowhere, so there is no byte-level identity check hiding behind an
address either.

## What the tests pin

1. **Value, not provenance.** Two separately generated `Address` objects that
   carry the same underlying bytes compare equal, and an address survives the
   `Val`/`ScAddress` round trip the storage layer performs without changing
   identity.
2. **Distinct addresses stay distinct**, including ones chosen to be
   confusable by a prefix or a text form — and `flag_dispute` accepts a party
   presented as a *different object* than the one stored, which is precisely
   what would break if identity were derived from a string.
3. **Reassignment moves identity with the value**: after `reassign_buyer`, the
   new party is recognized and the old one is not.
4. **`require_once_each` dedups by value**, not by argument position.
5. **A source-level guard**: `lib_source_never_stringifies_addresses` reads
   `src/lib.rs` and fails if any string-conversion token ever appears in it
   again, while asserting that the two legitimate comparison expressions are
   still present. Same style as the repo's existing
   `.github/scripts/check_error_discriminants.sh` guard, scoped to this issue's
   specific risk.

## Residual risk worth naming

`Address` deliberately does not implement `Ord`/`Hash` and has no text form in
`core`, which is why the audit is short. The one way this could regress is the
one the guard covers: someone adding a `strkey`-based comparison (for logging,
then reuse) — see `docs/dev-setup-troubleshooting.md` for how strkeys are meant
to be used for display only.
