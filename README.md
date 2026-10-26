# Royalty Rail

**A new settlement rail for royalties: every statement line proven, split and paid in one call.** Built on Stellar (Soroban) for HackMeridian 2026.

| | |
|---|---|
| **Live writer page** | https://royaltyrail-jcebongp.manus.space |
| **Demo video** | [`docs/demo/royalty-rail-demo.mp4`](docs/demo/royalty-rail-demo.mp4) (2:38) |
| **Contract (testnet)** | [`CDLROFPLMWIR4K2OCG2YV75ESGKO2UN4MR3FCGI33I7P4CA3IYI4NAUG`](https://stellar.expert/explorer/testnet/contract/CDLROFPLMWIR4K2OCG2YV75ESGKO2UN4MR3FCGI33I7P4CA3IYI4NAUG) |
| **Demo token (EURD, testnet)** | `CBOEK7O5XG35JY2EVZWMOZPLYJWLC3UTXUJAR73N4KFPJPVU6VOXKRAU` |
| **Writer release tx** | [`93de2abd…c6ced58e`](https://stellar.expert/explorer/testnet/tx/93de2abd98552559d2d195bf444c852081f124c6736d17eeaf89350fc6ced58e) |
| **Tests / build** | 31/31 passing · optimised WASM 34,058 bytes ([`artifacts/royalty_rail.wasm`](artifacts/royalty_rail.wasm)) |

Testnet and demo assets only. EURD is a test token, not EURC, and no real funds are involved.

> "My statement said paid. The money never came. Now every line proves itself."

Collecting societies move about €14bn a year to creators through a network of 228 societies in 111 countries, on statements and bank wires. Writers can't check that a line marked "paid" was actually paid, and money can be held without notice for as long as the society likes. Royalty Rail settles a statement on-chain in one call: each line is split across the registered rights holders, paid out in stablecoins, and given a receipt the writer can verify.

## Three rules a writer can rely on

1. **Splits are agreed, not imposed.** A work only becomes payable once every rights holder has accepted its splits. Changing them needs every old *and* new rights holder to accept, so nobody's share can be cut without their signature.
2. **No silent holds.** Withholding money needs a reason code (`DISPUTE`, `MISSING_DATA`, `LEGAL_ORDER`, `TAX_DOCS`, `IDENTITY`), a note, and an expiry of at most 90 days. The hold is published as an event addressed to the writer, and the withheld share sits in escrow, visible on-chain.
3. **No indefinite holds.** Once a hold expires, anyone (usually the writer) can release the escrowed money to the writer. The society's signature isn't needed. A society can renew a hold before it expires (each renewal max 90 days, published to the writer), but can't revive an expired one. A hold only catches that society's own statements.

And one rule for everyone else on the statement: **one writer's problem can't block anyone else's pay.** If a payout can't be delivered (for example, the writer's account has no trustline yet), it's credited to them in the rail and they claim it once their account is fixed. Everyone else is paid as normal.

## What's in here

```
contracts/royalty-rail/src/lib.rs    the contract
contracts/royalty-rail/src/test.rs   31 tests: the demo story, plus the review checks below
demo/run-demo.sh                     full demo on Stellar testnet
demo/statement-2027Q3.json           demo usage report (3 lines, 3 territories)
demo/statement-2027Q4.json           second statement, settled during a hold
demo/stage.sh                        fresh 2-minute hold for a live stage demo
web/                                 writer statement page (Vite + @stellar/stellar-sdk)
validation/                          test, build, testnet and integration logs
artifacts/royalty_rail.wasm          optimised contract build
docs/demo/                           demo video and voiceover script
```

## Quick start

```bash
cargo test                      # 31 tests, runs in under a second
stellar contract build          # -> target/wasm32v1-none/release/royalty_rail.wasm
./demo/run-demo.sh              # deploys and runs the whole story on testnet (needs jq)
```

## Contract API

| Function | Who signs | What it does |
|---|---|---|
| `__constructor(admin)` | — | Set at deploy |
| `add_society(society, name)` | admin | Onboard a society that may settle and hold |
| `add_token(token)` | admin | Allowlist a settlement token (e.g. EURC, USDC) |
| `upgrade(new_wasm_hash)` | admin | Ship a fix without redeploying (see notes) |
| `register_work(registrant, work_id, title, shares)` | registrant | Register an ISWC and propose its first splits (basis points, must sum to 10,000) |
| `propose_splits(proposer, work_id, shares)` | a current rights holder | Propose new splits; old splits stay in force until everyone accepts |
| `accept_splits(payee, work_id)` | payee | Accept pending splits; the last acceptance activates them |
| `settle(society, token, statement_id, lines)` | society | One transfer of the statement total into the rail, then payouts per payee, a receipt per (line, payee), escrow for held shares, credit for undeliverable payouts. Statement ids are unique. Max 25 lines / 75 receipts per call; split bigger reports into parts |
| `claim_credit(payee, token)` | **anyone** | Deliver a payee's credited payouts once their account can receive |
| `place_hold(society, payee, work_id?, reason, note, duration_secs)` | society | Hold one work or all of a payee's works, for this society's statements; returns the hold id. Max 5 holds in force per payee |
| `renew_hold(hold_id, extra_secs, note)` | society that placed it | Extend an unexpired hold (max 90 days per renewal) |
| `release_hold(hold_id)` | society that placed it | Pay the escrow (every currency) to the writer and close the hold |
| `claim_expired(hold_id)` | **anyone** | After expiry, pay the escrow to the writer. Needs no signature at all |
| `verify_receipt(statement_id, line_index, payee, hash)` | — | Does this hash match what the chain recorded? |
| `get_work`, `get_proposal`, `get_statement`, `statement_receipts`, `get_receipt`, `get_credit`, `get_hold`, `holds_for(payee, start, limit)`, `hold_count`, `is_society`, `is_token`, `version` | — | Read-only views |

**Events** (the writer-facing feed): `SplitsProposed`, `SplitsAccepted`, `SplitsActive`, `PayeeSettled` (one per payee per statement: lines, paid, credited, held), `StatementSettled`, `CreditClaimed`, `HoldPlaced`, `HoldRenewed`, `HoldReleased` (`RELEASED` or `EXPIRED`). Payee is a topic on every writer-facing event, so a wallet can subscribe to just their own.

**Receipt hash** = sha256 over the XDR of (statement id, line index, the line, splits version, payee, amount, society, token, paid / held / credited status).

## The 90-second stage demo

1. **The problem (15s).** "My statement said *paid as directed*. The money never came, and my account was put on hold without notice. I'm a Mercury Prize-winning writer and I couldn't prove it."
2. **Settle (20s).** The UK society settles Q3: a gig in Lisbon, UK radio, streaming in Nigeria. Two writers and a publisher are paid in seconds. Show the receipts.
3. **Verify (15s).** Writer A pastes the hash from line 0 of their statement: `true`. Change one character: `false`.
4. **Hold (20s).** The society holds Writer B's share. It *has* to give a reason and an expiry, and Writer B sees it immediately. The next statement pays everyone else; B's share sits in escrow on-chain.
5. **The kicker (20s).** The hold expires. Writer B releases their own money, no society signature. "Your money can be held. It can't be held silently, and it can't be held forever."

Use generic names on stage ("UK society", "Portuguese society"), not real ones.

## Roadmap

In priority order (item 1, the writer statement view, is built: see `web/` and the live page):

1. **Writer statement view.** One page: paste an address, see each statement line with territory, source, amount, status (paid / held + reason + countdown to expiry) and a "verify" button. This is what judges will remember.
2. **Society console.** Upload a usage report JSON, settle, place a hold. Can be a thin form over the same contract calls as `run-demo.sh`.
3. **Cash-out.** Writer withdraws EURD/USDC to local currency through a testnet anchor (SEP-24). Even a mocked last step shows the end-to-end story.
4. **Stretch: confidential amounts.** Pay out with Stellar's confidential tokens so *who* was paid stays auditable but *how much* a writer earns stays private.

## Review against Stellar's Soroban guidance

The contract was checked against Stellar's Soroban skill (skills.stellar.org) on day 1 of the hack. What changed:

| Guidance | What we found | Fix | Test |
|---|---|---|---|
| Check trustlines before payments | One writer without a trustline would have cancelled the whole statement for everyone | Payouts use a caught transfer; failures are credited to the writer, claimable later | `writer_without_trustline_does_not_block_everyone_else`, `credited_payout_is_claimed_once_the_account_can_receive` |
| Validate token addresses against an allowlist | Any token could be used to "pay" | Admin allowlist (`add_token`) | `only_allowlisted_tokens_settle` |
| Avoid unbounded loops over user-controlled collections | Every settlement scanned a payee's entire hold history | Max 5 holds in force per payee; history paginated; holds loaded once per settlement; max 10 rights holders per work | `holds_in_force_are_capped_and_history_is_paginated`, `works_have_at_most_ten_rights_holders`, `worst_case_statement_fits_network_limits` |
| Extend TTL on active use, only below a threshold | Works were only extended when written, so a work paid every quarter could be archived; the instance only lived 30 days | Entries a settlement reads are extended once within 30 days of archival (to 120 days); same for the instance and code | `settling_keeps_the_work_and_contract_alive` |
| Don't rely on `mock_all_auths` | Signature trees were never checked | Tests assert the exact signatures: settle = one society signature covering one transfer; `claim_expired` = none; admin calls refuse non-admins | `settle_needs_exactly_one_society_signature_covering_one_transfer`, `claiming_an_expired_hold_needs_no_signature_at_all`, `admin_functions_reject_everyone_but_the_admin` |
| Verify event emission | Not tested | Writer-facing event checked | `the_writer_sees_the_hold_as_an_event_addressed_to_them` |
| Decide on upgradability | Undecided | Admin-guarded `upgrade` + `contractmeta`, for the hack | `admin_functions_reject_everyone_but_the_admin` |
| (found while reviewing) | A UK hold also caught money paid by the Portuguese society; settling in a second currency during a hold blocked the statement | Holds only catch their own society's statements; escrow is kept per currency | `a_hold_only_catches_its_own_societys_statements`, `a_hold_escrows_each_currency_separately` |

Still open, and worth one sentence each if a judge asks:

- **Work registration is open.** Anyone can register an ISWC, so someone could squat a real work with themselves as sole payee. In production a society would attest ownership (e.g. a society co-sign on `register_work`, or only settling works it has matched).
- **No admin rotation or society removal.** Fine for the hack; production needs both, behind a timelock.
- **`upgrade` is admin-only.** For production, put it behind a timelock or governance, or remove it to make the contract immutable.
- **Clawback.** If the settlement token is clawback-enabled, its issuer can claw back money escrowed in the rail. That's a property of the token, not the contract; say so if asked.
- **Any current rights holder can replace a pending splits proposal** (which resets acceptances). Harmless to money, but could be used to stall a change.
- **Not run through static analysis** (e.g. Scout Soroban), which the guidance recommends before mainnet.

## Development notes

- **Testnet and WASM build: done.** See *Tested deployment and writer page* below and `validation/` for the logs.
- **SDK version.** Pinned to `soroban-sdk = "28.0.0"`. If testnet is on a newer protocol and deploy complains, bump the version in the root `Cargo.toml`.
- **Size limits.** The largest statements the limits allow (25 lines; or 7 lines across a 10-way split with holds and a credit) were tested against network resource limits (ledger-entry size, event size, CPU). Bigger usage reports are settled in parts; `settle` refuses rather than fails on-chain.
- **Demo token.** `EURD` is a test asset issued by the admin key, standing in for EURC. Don't call it EURC on stage.
- **Holds in the demo** last 2 minutes so expiry can be shown live; the contract allows up to 90 days.

## Tested deployment and writer page — 25 October 2026

The checks above have now been run in a fresh environment: Rust stable 1.99.1, Stellar CLI 28.1.0, jq 1.7, all 31 tests passing, and an optimized `royalty_rail.wasm` of 34,058 bytes. Contract sources are unchanged. `Cargo.lock` records the resolved Rust dependencies.

`deployment.json` records the testnet contract, EURD token, account addresses, original statement IDs, and stage statements/holds. `demo/run-demo.sh` successfully settled Q3, held writer-b's Q4 share, and released 350 EURD after on-chain expiry. Current CLI optional `work_id` arguments must be JSON-quoted; the script handles that. It also pins the testnet RPC/passphrase, saves progress, and waits for ledger time rather than a fixed sleep. The testnet's unusually high storage rent required additional free Friendbot XLM; `demo/fund-testnet.sh` provides a bounded funding helper without changing contract TTL or logic.

### Run the live stage demo

On a new computer, run `./demo/import-demo-keys.sh` to import this deployment's generated identities. The private archive includes `demo/demo-keys.json` for this purpose. Keep that file private, never publish it, and never use those keys on mainnet or with real funds. The importer refuses to overwrite a different identity with the same alias.

Run `./demo/stage.sh` immediately before going on stage. It creates a fresh 120-second MISSING_DATA hold and a unique settlement, leaving the claim for the writer page. If there is already an unexpired hold, it refuses to create a misleading demonstration; wait for expiry and run it again. It saves the new IDs and expiry to deployment.json.

The `web/` directory contains the single-page statement app. Use Node 22.12+ and `corepack pnpm install --frozen-lockfile`, then `corepack pnpm dev` from web/. The public page reads the indexed `payee_settled` events, so fresh stage statements appear without a website republish. Use Refresh or its automatic 20-second refresh. The deployment file provides a fallback for the known demo statements.

Select Writer A or Writer B, expand a Payment receipt, and verify it. Toggle Tamper with hash and verify again to show rejection. For release, expand Release signing and paste only writer-b's generated testnet key from demo/demo-keys.json, or connect Freighter on testnet. The published site contains no private keys; entered keys stay in browser memory only.

Receipt hashes describe the original settlement outcome. A released held receipt still verifies as the original Held record, while the page displays its current Paid state with an explanation. Credited receipts show that delivery failed at settlement; they do not claim to prove a later withdrawal.
