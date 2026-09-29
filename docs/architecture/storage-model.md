# Storage model

How the `shade` contract (`contracts/shade/`) partitions its Soroban storage: the key enums, the storage tier each key lives in, ID-counter and collection patterns, and TTL/rent behavior as actually implemented — not the Soroban platform defaults in the abstract.

## Storage key enums

Soroban hard-caps every `#[contracttype]` enum at 50 cases. `contracts/shade/src/types.rs` partitions storage keys into one `#[contracttype]` enum per feature domain instead of one monolithic key enum — see the module-level doc comment at [`contracts/shade/src/types.rs:1-24`](../../contracts/shade/src/types.rs#L1-L24). Each enum is independent, so identical variant names in different enums (e.g. `Campaign` appears in both [`CampaignKey`](#campaignkey) and [`BackerKey`](#backerkey)) never collide — Soroban scopes storage by the full typed key, enum included.

| Enum | Domain | Defined in |
|---|---|---|
| `DataKey` | Core payment engine: admin, merchants, invoices, subscriptions, fees, analytics, escrow, auto-withdrawal | [`types.rs:32-84`](../../contracts/shade/src/types.rs#L32-L84) |
| `EventKey` | Ticketing / events | [`types.rs:89-98`](../../contracts/shade/src/types.rs#L89-L98) |
| `CampaignKey` | Crowdfunding campaigns, categories, tags, fee-policy campaigns, pledge campaigns, comments, vesting, hard-cap voting, leaderboard | [`types.rs:109-167`](../../contracts/shade/src/types.rs#L109-L167) |
| `BackerKey` | Backer reward tiers and perks | [`types.rs:172-181`](../../contracts/shade/src/types.rs#L172-L181) |
| `StretchKey` | Stretch-goal milestones | [`types.rs:186-196`](../../contracts/shade/src/types.rs#L186-L196) |
| `VestingKey` | Creator fund vesting | [`types.rs:204-213`](../../contracts/shade/src/types.rs#L204-L213) |
| `FiatGoalKey` | Fiat-pegged campaign goals | [`types.rs:221-232`](../../contracts/shade/src/types.rs#L221-L232) |
| `AnalyticsKey` | Campaign analytics exports | [`types.rs:240-258`](../../contracts/shade/src/types.rs#L240-L258) |
| `NftKey` | NFT reward collections | [`types.rs:263-271`](../../contracts/shade/src/types.rs#L263-L271) |
| `GovKey` | DAO governance | [`types.rs:276-283`](../../contracts/shade/src/types.rs#L276-L283) |
| `BridgeKey` | Cross-chain bridge deposits and pledges | [`types.rs:288-301`](../../contracts/shade/src/types.rs#L288-L301) |
| `MultiSigKey` | Multi-sig massive-withdrawal proposals | [`types.rs:306-319`](../../contracts/shade/src/types.rs#L306-L319) |

This page documents [`DataKey`](#datakey-core-payment-engine) in full, since it is the core payment-engine enum that [`docs/contracts/shade.md`](../contracts/shade.md), [`docs/concepts/refunds-and-voids.md`](../concepts/refunds-and-voids.md), and [`docs/concepts/merchants.md`](../concepts/merchants.md) all depend on. The other eleven enums (ticketing, crowdfunding, governance, bridge, multi-sig, etc.) are out of scope for this page; they are listed above for orientation only.

## `DataKey` (core payment engine)

Every variant in [`contracts/shade/src/types.rs:32-84`](../../contracts/shade/src/types.rs#L32-L84), the value stored under it, and the component(s) that read or write it. All `DataKey` entries live in **persistent storage** — see [Storage tier usage](#storage-tier-usage-in-this-contract) below for why.

| Variant | Value type | Owning component | Read/write sites |
|---|---|---|---|
| `ContractInfo` | `ContractInfo { admin, timestamp }` | `shade.rs::initialize` | Written once at `initialize`; not read elsewhere in the components read for this page. |
| `Admin` | `Address` | `core.rs` | Written by `initialize`, `accept_admin_transfer`. Read by `core::get_admin`, `core::assert_admin` (used throughout admin-gated calls). |
| `PendingAdmin` | `Address` | `admin.rs` | Written by `propose_admin_transfer`, removed by `accept_admin_transfer`. Read by `accept_admin_transfer`. |
| `Paused` | `bool` | `pausable.rs` | Written by `pause`/`unpause`. Read by `is_paused`, `assert_paused`, `assert_not_paused` (called at the top of most state-changing trait methods). |
| `AcceptedTokens` | `Vec<Address>` | `admin.rs` | Written by `add_accepted_token(s)`/`remove_accepted_token`. Read by `is_accepted_token` and everywhere token acceptance is checked. |
| `ReentrancyStatus` | `bool` (presence-based) | `reentrancy.rs` | Set by `reentrancy::enter`, removed by `reentrancy::exit`. See [Reentrancy guard](#reentrancy-guard). |
| `AccountWasmHash` | `BytesN<32>` | `admin.rs`, read by `account_factory.rs` | Written by `set_account_wasm_hash`. Read by `account_factory::deploy_account`; panics with `WasmHashNotSet` if absent. |
| `PlatformAccount` | `Address` | `admin.rs` | Written by `set_platform_account`. Read by `get_platform_account` (defaults to the current admin if unset) and every payment-split path. |
| `Role(Address, Role)` | `bool` (presence-based) | `access_control.rs` | Set by `grant_role`, removed by `revoke_role`. Read by `has_role`/`assert_has_role`. |
| `UsedNonce(Address, BytesN<32>)` | — | `signature_util.rs` (not read for this page) | Nonce replay guard for `create_invoice_signed`. |
| `FeeInBasisPoints(Address)` | `i128` | — | Declared in the enum; not written or read by any component read for this page. |
| `FeeAmount(Address)` | `i128` | — | Declared in the enum; not written or read by any component read for this page. |
| `TokenFee(Address)` | `i128` | `admin.rs` | Written by `set_fee`, `execute_fee`. Read by `get_fee`, and as the fallback in `platform_fee::effective_fee_bps` when no per-merchant override exists. |
| `PendingTokenFee(Address)` | `PendingFee { token, fee, proposed_at }` | `admin.rs` | Written by `propose_fee`, removed by `execute_fee`. Read by `get_pending_fee`, `execute_fee`. |
| `MerchantPlatformFee(u64, Address)` | `i128` | `platform_fee.rs` | Written by `set_merchant_platform_fee`, removed by `clear_merchant_platform_fee`. Read by `get_merchant_platform_fee` and `effective_fee_bps` as the per-merchant override. |
| `TokenOracle(Address)` | `OracleConfig` | `admin.rs` | Written by `set_token_oracle`. Read by `get_token_oracle`; panics with `OracleNotConfigured` if absent. |
| `Merchant(u64)` | `Merchant` record | `merchant.rs` | Written by `register_merchant`, `set_merchant_status`, `verify_merchant`, `set_merchant_webhook`. Read by `get_merchant` and everywhere a merchant record is needed. |
| `MerchantKey(Address)` | `BytesN<32>` | `merchant.rs` | Written by `set_merchant_key`. Read by `get_merchant_key`, `signature_util` for signed-invoice verification. |
| `MerchantCount` | `u64` | `merchant.rs` | See [ID counters](#id-counter-pattern). |
| `MerchantId(Address)` | `u64` | `merchant.rs` | Address → merchant-ID reverse index. Written at registration. Read by `is_merchant`, `get_merchant_id`, `find_merchant_id`, and everywhere an address must resolve to a merchant ID. |
| `MerchantTokens(Address)` | `Vec<Address>` | `merchant.rs` | Per-merchant accepted-token allowlist. See [Collection-shaped storage](#collection-shaped-storage). |
| `MerchantBalance(Address)` | `i128` | — | Declared in the enum; not written or read by any component read for this page. |
| `MerchantAccount(u64)` | `Address` | `merchant.rs` | Written by `set_merchant_account`. Read by `get_merchant_account`, `restrict_merchant_account`, and every payment/refund path that needs the merchant's fund-holding account. |
| `Invoice(u64)` | `Invoice` record | `invoice.rs` | Written by every invoice-creating and invoice-mutating function. Read by `get_invoice` and everywhere an invoice is looked up. |
| `InvoiceCount` | `u64` | `invoice.rs` | See [ID counters](#id-counter-pattern). |
| `SubscriptionPlan(u64)` | `SubscriptionPlan` record | `subscription.rs` (not read in full for this page) | — |
| `Subscription(u64)` | `Subscription` record | `subscription.rs` | — |
| `PlanCount` | `u64` | `subscription.rs` | ID counter for subscription plans. |
| `SubscriptionCount` | `u64` | `subscription.rs` | ID counter for subscriptions. |
| `MerchantVolume(Address, Address)` | `i128` | `admin.rs` | Legacy per-merchant/token volume figure, kept in sync alongside `MerchantAnalytics`. Read as a fallback inside `get_merchant_analytics` when no `MerchantAnalytics` entry exists yet. |
| `UserTransactions(Address)` | — | `history.rs` (not read in full for this page) | Per-user transaction history. |
| `MerchantAnalytics(Address, Address)` | `MerchantAnalytics` record | `admin.rs` | Written by `record_merchant_payment`. Read by `get_merchant_analytics`, `get_merchant_volume`. |
| `MerchantAnalyticsSummary(Address)` | `MerchantAnalyticsSummary` record | `admin.rs` | Written by `record_merchant_payment`. Read by `get_merchant_analytics_summary`. |
| `TokenAnalytics(Address)` | `TokenAnalytics` record | `admin.rs` | Written by `record_token_payment`. Read by `get_token_analytics`. |
| `TokenVolume(Address)` | `i128` | `admin.rs` | Written by `record_token_payment`. Read by `get_token_volume`, `get_token_dominance_metrics`, `get_top_tokens_by_volume`, `get_token_market_share`. |
| `MerchantAutoWithdrawalThreshold(u64, Address)` | `i128` | `auto_withdrawal.rs` (not read in full for this page) | — |
| `MerchantAutoWithdrawalRecipient(u64)` | `Address` | `auto_withdrawal.rs` (not read in full for this page) | — |
| `Escrow(u64)` | `Escrow` record | `escrow.rs` (not read in full for this page) | — |
| `EscrowCount` | `u64` | `escrow.rs` | ID counter for escrows. |

> **Note:** `FeeInBasisPoints`, `FeeAmount`, and `MerchantBalance` are declared as `DataKey` variants but were not written or read by any of the components inspected for this page (`admin.rs`, `merchant.rs`, `invoice.rs`, `platform_fee.rs`, `pausable.rs`, `reentrancy.rs`, `access_control.rs`, `core.rs`). Treat them as reserved/unused in the current implementation rather than active storage until a future page verifies otherwise.

## Storage tier usage in this contract

Soroban offers three storage durability tiers (platform behavior, not project-specific):

- **Instance storage** — small, tied to the contract instance's own TTL; typically used for a handful of contract-wide config values.
- **Persistent storage** — long-lived, rent-paying, archived (and inaccessible) once its TTL expires unless extended.
- **Temporary storage** — cheap, expires quickly, and is unsuitable for anything that must survive past a short window.

**Observed usage in `contracts/shade/src/components/` and `contracts/shade/src/shade.rs`: every storage call in production code goes through `env.storage().persistent()`.** There is no `.instance()` or `.temporary()` call anywhere in `contracts/shade/src/components/*.rs` or `contracts/shade/src/shade.rs`. The only `.instance()` calls in the crate are in test-only mock contracts under `contracts/shade/src/tests/` (e.g. `test_escrow_integration.rs`, `test_fiat_goals.rs`, `test_fiat_pricing.rs`, `test_feature_201.rs`), which use instance storage to hold mock price/state for a test double, not as part of the `shade` contract's own storage model.

**Project-specific storage-tier selection rule (derived from actual usage, not a general Soroban recommendation):** this contract puts everything — admin config, merchant records, invoices, fee schedules, analytics, counters, reentrancy/pause flags — in persistent storage, uniformly. There is no code-level distinction between "config that should live as long as the contract" (a natural instance-storage candidate, e.g. `Admin`, `PlatformAccount`, `Paused`) and "records that could reasonably expire" (a natural temporary-storage candidate, e.g. `UsedNonce`). Document any change to this pattern here if a future PR introduces instance or temporary storage for production keys, since it would be a deviation from the uniform-persistent approach this page currently describes.

## ID-counter pattern

Every counted collection in `DataKey` (merchants, invoices, subscription plans, subscriptions, escrows) follows the same pattern, verified directly in `merchant.rs` and `invoice.rs`:

| Counter | Counter key | Value type | First ID | Increment timing |
|---|---|---|---|---|
| Merchants | `DataKey::MerchantCount` | `u64` | `1` | Read, incremented by 1, and written back inside `register_merchant` before the new `Merchant` record is stored — see [`merchant.rs:25-53`](../../contracts/shade/src/components/merchant.rs#L25-L53). |
| Invoices | `DataKey::InvoiceCount` | `u64` | `1` | Same pattern inside `create_invoice`, `create_fiat_invoice`, `create_invoice_draft`, `create_invoice_signed` — see e.g. [`invoice.rs:146-173`](../../contracts/shade/src/components/invoice.rs#L146-L173). |
| Subscription plans | `DataKey::PlanCount` | `u64` | — | Not read in full for this page; declared alongside the same-shaped `SubscriptionCount`. |
| Subscriptions | `DataKey::SubscriptionCount` | `u64` | — | Not read in full for this page. |
| Escrows | `DataKey::EscrowCount` | `u64` | — | Not read in full for this page. |

**Mechanics, verified from `merchant.rs`/`invoice.rs`:**

```rust
let merchant_count: u64 = env
    .storage()
    .persistent()
    .get(&DataKey::MerchantCount)
    .unwrap_or(0);
let new_id = merchant_count + 1;
// ... store the record under DataKey::Merchant(new_id) ...
env.storage().persistent().set(&DataKey::MerchantCount, &new_id);
```

- **Monotonicity:** the counter only ever increases by 1 per creation; there is no decrement path in `merchant.rs` or `invoice.rs`.
- **Start value:** the counter defaults to `0` via `.unwrap_or(0)` when absent, so the first ID issued is `1`. ID `0` is never assigned and is used as a sentinel by callers (e.g. `merchant::get_merchant` panics with `MerchantNotFound` if `merchant_id == 0`).
- **Reuse:** IDs are never reused — there is no delete/free-list mechanism for merchants or invoices in the components read for this page.
- **Gaps:** none possible from the counter mechanics themselves, since every increment is paired with a record write in the same call. (This page does not verify atomicity guarantees beyond what Soroban's own transaction model provides.)
- **Overflow:** `u64` counters; no explicit overflow check exists in `merchant.rs` or `invoice.rs`. Rust's default (in a `#![no_std]`, non-test build) is a wrapping or panicking add depending on build profile — this page does not verify which, and does not claim overflow is handled.

## Collection-shaped storage

Two patterns exist for storage that grows with usage, verified in `merchant.rs`:

- **`DataKey::MerchantTokens(Address)`** — `Vec<Address>`, one entry per merchant, holding that merchant's accepted-token allowlist. `set_merchant_accepted_tokens` rewrites the whole vector (deduplicated) on every call; `remove_merchant_accepted_token` reads the vector, filters out one entry, and writes the result back — see [`merchant.rs:320-401`](../../contracts/shade/src/components/merchant.rs#L320-L401). There is no cap on the number of tokens a merchant can add in the code read for this page.
- **Merchant/invoice listing** (`get_merchants`, `get_invoices`) — these do not use a stored collection key at all. They loop `1..=count` (the current `MerchantCount`/`InvoiceCount`) and issue one `get` per ID, filtering in memory — see [`merchant.rs:219-255`](../../contracts/shade/src/components/merchant.rs#L219-L255) and [`invoice.rs:516-574`](../../contracts/shade/src/components/invoice.rs#L516-L574). This means every call to `get_merchants`/`get_invoices` costs one storage read per merchant/invoice that has ever been created, growing linearly and unboundedly with the total count — there is no pagination in these two functions specifically. (The trait separately exposes `search_invoices_paginated` and `search_merchants_paginated`, declared in `shade_interface.rs`, which this page does not trace in detail.)

> **Warning:** `get_merchants` and `get_invoices` read every merchant/invoice ever created on every call. As the counters grow, these calls become progressively more expensive (CPU and possibly resource-limit risk on Soroban). This is an observed characteristic of the current implementation, not a documented protocol limit — no cap exists in the code read for this page.

There is no documented or enforced upper bound on `MerchantCount`, `InvoiceCount`, or `MerchantTokens(Address)` length in the components read for this page.

## TTL and rent

**No `extend_ttl`, `bump_ttl`, or equivalent TTL-extension call exists anywhere in `contracts/shade/src/`** (verified by search across `contracts/shade/src/`, production and test code). Every entry this contract writes goes into persistent storage (see [Storage tier usage](#storage-tier-usage-in-this-contract)) with no code-level renewal.

This has the following implications, labeled by their source:

- **Platform behavior (Soroban, not this project):** persistent storage entries accrue rent and are archived once their TTL expires; an archived entry cannot be read until it is restored (subject to Soroban's state-archival/restoration rules).
- **Observed behavior (this codebase):** since the contract never calls an extension function, an entry's TTL is whatever Soroban assigns by default at write time and is never proactively renewed by contract logic.
- **Interpretation (not stated anywhere in code or existing docs):** long-lived records that are read infrequently after creation — e.g. an old `Invoice(u64)` for a fully paid and refunded invoice, or a `Merchant(u64)` for a merchant that stopped transacting — are the entries most exposed to eventual TTL expiration under Soroban's platform rules, since nothing in this contract refreshes them.
- **Operational recommendation (not implemented in code):** an operator relying on long-term on-chain queryability of old invoices, subscriptions, or merchant records should track TTL/rent externally (e.g. via a keeper that calls Soroban's restore/bump host operations directly, outside this contract) since the contract itself provides no mechanism to do so.

**This page makes no claim about what happens to Shade's specific data (invoices, subscriptions, merchant records) once a persistent entry's TTL actually expires**, beyond the general Soroban archival behavior above — no test or component read for this page exercises or asserts expired-entry behavior, and no code path in `contracts/shade/src/components/` checks or handles a "was this entry restored/archived" condition. Do not assume an expired invoice or subscription is silently deleted, silently preserved, or automatically recoverable; none of those is verified here.

## Related pages

- [Shade contract reference](../contracts/shade.md)
- [Refunds and voids](../concepts/refunds-and-voids.md)
- [Merchants](../concepts/merchants.md)
- [Glossary: Instance / Persistent / Temporary Storage](../glossary.md#instance--persistent--temporary-storage)
- [Glossary: TTL / Rent](../glossary.md#ttl--rent)
