# The component pattern

How business logic is organized inside the `shade` contract, and the rules that keep 34 independently-owned modules from turning into a tangle of cross-calls. This page is for anyone adding or reviewing a feature under `contracts/shade/src/components/`.

## The three-file pattern

Every feature in `shade` is split across three files with a fixed division of labor:

1. **A component module** under `contracts/shade/src/components/` — the actual logic: storage reads/writes, validation, authorization checks specific to the feature, and event emission. Components are plain Rust modules (`pub fn`, no `#[contract]` or `#[contractimpl]`); they are never invoked directly by a client.
2. **`shade_interface.rs`** — the single `#[contracttrait] pub trait ShadeTrait` that is the contract's complete public surface. It declares every exported function's signature (parameter order, types, return type) with no bodies.
3. **`shade.rs`** — the `#[contractimpl] impl ShadeTrait for Shade` block: one method per trait function, each just wide enough to apply cross-cutting guards (mainly `pausable_component::assert_not_paused`) and then delegate to the matching component function.

> **Note:** The repository also contains a file named `contracts/shade/src/interface.rs`. It declares a second, overlapping `ShadeTrait`-like definition but is **not** listed in [`contracts/shade/src/lib.rs`](../../contracts/shade/src/lib.rs)'s module declarations, so it is never compiled and has no effect on the built contract. It appears to be left over from an earlier version of the interface before it was renamed/consolidated into `shade_interface.rs`, and it has drifted out of sync (it's missing several functions `shade.rs` implements, such as everything in the pledge/campaign/vesting/fiat-goal/analytics families, and has stray syntax left over from a merge). A handful of existing docs (`docs/glossary.md`, `docs/reference/data-types.md`) still cite line numbers in `interface.rs` — those citations point at dead code. Treat `shade_interface.rs` as the only real interface file; this page and the other three added alongside it cite `shade_interface.rs` throughout.

## Conventions every component follows

- **`&Env` is always the first parameter.** Components take `env: &Env` (a borrow), not an owned `Env` — `shade.rs` owns the `Env` value and hands out borrows to every component call in a method.
- **Auth is asserted inside the component, not in `shade.rs`.** `shade.rs` methods do not call `require_auth()` themselves (with a couple of narrow exceptions like `emit_bridge_placeholder` and `record_campaign_contribution`, which authorize the caller inline before delegating). The `Address.require_auth()` call and any role/ownership check lives at the top of the component function, so the same authorization logic runs whether the function is reached through `ShadeTrait` or called directly in a test.
- **Failure is `panic_with_error!`, not `Result`.** No function in a component returns `Result<T, E>`. Every error path calls `panic_with_error!(env, SomeError::Variant)`, which aborts the whole invocation and rolls back all state changes made so far in the transaction — see [Cross-Contract Calls: Failure Semantics](cross-contract-calls.md#failure-semantics--atomic-guarantees). There is no partial-success return value to check.
- **Storage access goes through the owning component's helpers.** A component reads and writes only the `DataKey`/`EventKey`/`CampaignKey`/… variants that belong to its feature (see [Storage and layering](#storage-and-layering) below). Other components read that state through public functions the owning component exports (e.g. `merchant::get_merchant_id`, `admin::is_accepted_token`), not by reaching into storage directly.
- **Events are emitted after the state write that makes them true**, generally as the last step of the function (occasionally followed only by a call into another component, such as `history::record_transaction` or `auto_withdrawal::check_and_trigger_auto_withdrawal`, that itself emits further events). A component never emits an event and then goes on to a step that can still panic.
- **Read-only query functions still validate their input** (existence, e.g. `merchant::get_merchant` panics with `MerchantNotFound` for an unknown ID) but never emit events or require auth, except where a query is deliberately telemetry-generating (see `search.rs`'s `*_search_executed_event`s).

## Worked example: `pause`

`pause` is one of the smallest complete slices in the codebase and touches all three files without any of the extra machinery (fee routing, cross-contract calls) that makes bigger examples noisy.

**1. Component (`contracts/shade/src/components/pausable.rs`):**

```rust
pub fn pause(env: &Env, admin: &Address) {
    admin.require_auth();

    if core::get_admin(env) != admin.clone() {
        panic_with_error!(env, ContractError::NotAuthorized);
    }

    assert_not_paused(env);

    env.storage().persistent().set(&DataKey::Paused, &true);

    events::publish_contract_paused_event(env, admin.clone(), env.ledger().timestamp());
}
```

Auth, the admin check, the state guard (`assert_not_paused` — you can't pause an already-paused contract), the storage write, then the event. Everything the function needs to be correct on its own, independent of how it's reached.

**2. Interface (`contracts/shade/src/shade_interface.rs`):**

```rust
fn pause(env: Env, admin: Address);
```

Just the signature, as one line among ~180 others in the trait.

**3. Wiring (`contracts/shade/src/shade.rs`):**

```rust
fn pause(env: Env, admin: Address) {
    pausable_component::pause(&env, &admin);
}
```

No guard in front of `pause` itself — pausing has nothing to pause-guard against — just the delegation. Contrast with a state-changing function like `add_accepted_token`, which is guarded:

```rust
fn add_accepted_token(env: Env, admin: Address, token: Address) {
    pausable_component::assert_not_paused(&env);
    admin_component::add_accepted_token(&env, &admin, &token);
}
```

That one extra line — `pausable_component::assert_not_paused(&env)` — is the entire job of `shade.rs` for the ~150 state-changing methods that need it. `shade.rs` is deliberately a thin wiring layer: if a `shade.rs` method body is more than a guard call plus a delegation, that logic almost always belongs in the component instead.

```mermaid
flowchart LR
    Client -->|pause(admin)| ShadeTrait["shade_interface.rs\nShadeTrait::pause"]
    ShadeTrait -.signature only.-> Shade["shade.rs\nimpl ShadeTrait for Shade"]
    Shade -->|delegates| Pausable["components/pausable.rs\npause()"]
    Pausable -->|require_auth, assert_admin, assert_not_paused| Pausable
    Pausable -->|storage write| Storage[(DataKey::Paused)]
    Pausable -->|emit| Event[[ContractPausedEvent]]
```

A guarded example follows the same shape with one extra hop: `shade.rs` calls `pausable_component::assert_not_paused(&env)` before delegating to the target component, so the pause check runs once, centrally, rather than being duplicated inside every component.

## Component inventory

Every module declared in [`contracts/shade/src/components/mod.rs`](../../contracts/shade/src/components/mod.rs) — 34 in total, not the 15 the original tracking issue described (the issue text was written against an earlier state of the repository and is stale).

| Module | Responsibility | Key public functions | Related concept doc |
|---|---|---|---|
| `access_control` | Grants/revokes/checks `Role` (Admin/Manager/Operator) beyond the single contract admin | `grant_role`, `revoke_role`, `has_role`, `assert_has_role` | — |
| `account_factory` | Deploys a per-merchant `account` contract instance from the stored WASM hash | `deploy_account` | [Upgradeability §`set_account_wasm_hash`](upgradeability.md#set_account_wasm_hash) |
| `admin` | Accepted-token whitelist, per-token fees (incl. time-locked updates), platform account, oracle config, admin transfer, token/merchant volume analytics | `add_accepted_token`, `set_fee`, `propose_fee`/`execute_fee`, `calculate_fee`, `propose_admin_transfer`/`accept_admin_transfer` | — |
| `analytics` | Running per-campaign contribution aggregates and immutable analytics export snapshots | `export_campaign_analytics`, `get_campaign_stats` | — |
| `auto_withdrawal` | Per-merchant/token sweep threshold; triggers a withdrawal to a configured recipient once a merchant account's balance crosses it | `set_auto_withdrawal_threshold`, `check_and_trigger_auto_withdrawal` | [Auto-withdrawal and merchant settlement](../concepts/auto-withdrawal.md) |
| `backer_rewards` | Tiered crowdfunding rewards: pledge tiers, perks, fulfillment | `create_backer_campaign`, `pledge_to_campaign`, `select_backer_reward_tier`, `claim_backer_perk` | — |
| `bridge` | Registers bridge listeners (relayers) and records de-duplicated external-chain deposits | `register_bridge_listener`, `record_bridge_deposit` | — |
| `campaign` | Fee-waiver/discount/staking/affiliate promotional campaigns (distinct from the category/tag campaigns in `campaigns`) | `create_campaign`, `stake_campaign`, `register_affiliate`, `pay_affiliate_commission` | — |
| `campaign_royalties` | Royalty configuration and payout on secondary sales tied to a campaign | `set_campaign_royalty` | — |
| `campaigns` | Categories, tags, and the primary `Campaign` record (title/goal/token/deadline) | `create_campaign_category`, `create_campaign`, `record_contribution`, `get_campaigns` | — |
| `comments` | Backer comments and moderation (flag/remove) on a crowdfund | `create_comment` | — |
| `core` | The one thing every other component depends on: reading the admin address and asserting a caller is the admin | `get_admin`, `assert_admin` | — |
| `cross_chain_pledge` | Records inbound cross-chain pledges and tracks their status | `create_pledge` | — |
| `escrow` | Buyer/seller-funded token holding tied to an optional invoice | `create_escrow`, `fund_escrow`, `release_escrow`, `refund_escrow` | [Escrow](../concepts/escrow.md) |
| `event` | Event ticketing: creation, dynamic (early-bird/late-markup) pricing, purchase, resale, batch refund on cancellation | `create_event`, `purchase_ticket`, `configure_dynamic_pricing`, `resell_ticket` | [Event ticketing, dynamic pricing, and resale](../concepts/event-ticketing.md) |
| `fiat_goals` | Fiat-pegged campaign funding targets, valued through a token's price oracle | `set_campaign_fiat_goal`, `record_fiat_contribution`, `refresh_campaign_fiat_quote` | [Fiat pricing and oracles](../concepts/fiat-pricing-and-oracles.md) (covers the invoice side of the same oracle mechanism) |
| `governance` | DAO council membership, voting config, and vote-gated contract upgrades | `add_gov_member`, `propose_upgrade`, `vote_on_upgrade`, `finalize_upgrade` | [Upgradeability §the governance path](upgradeability.md#the-governance-path) |
| `history` | Append-only per-user transaction log | `record_transaction`, `get_user_transactions` | [Merchant analytics and transaction history](../concepts/analytics-and-history.md) |
| `invoice` | The core payment object: creation (crypto/fiat/draft/signed), payment (full/partial/batch), refund (full/partial), void, amend | `create_invoice`, `pay_invoice`, `pay_invoice_partial`, `refund_invoice`, `resolve_invoice_amount` | [Fiat pricing and oracles](../concepts/fiat-pricing-and-oracles.md) |
| `leaderboard` | Per-campaign top-donor tracking | `init_campaign`, `track_donation`, `get_top_donors` | — |
| `merchant` | Merchant registration, activation/verification, account/webhook/accepted-token settings | `register_merchant`, `is_merchant_active`, `set_merchant_account`, `set_merchant_accepted_tokens` | — |
| `multisig_withdrawal` | Threshold-gated withdrawal proposals requiring a quorum of registered signers | `propose_withdrawal`, `approve_withdrawal`, `cancel_withdrawal` | — |
| `nft` | Reward NFT collections: mint, batch mint, transfer, burn, claim | `create_nft_collection`, `mint_nft`, `claim_nft_reward` | — |
| `pausable` | The global emergency pause flag and its guard, checked by nearly every state-changing entry point | `pause`, `unpause`, `is_paused`, `assert_not_paused` | [Pausable](../security/pausable.md) |
| `payment` | Validates a `PaymentPayload` (direct vs. swap route, slippage bounds) | `validate_payment_payload` | [Payment payloads, swap routing, and the cross-chain bridge placeholder](../concepts/payment-payloads-and-routing.md) |
| `platform_fee` | Computes the merchant/platform fee split for a payment and executes the split token transfers | `compute_split`, `route_from_payer`, `route_from_allowance`, `set_merchant_platform_fee` | — |
| `pledge` | All-or-nothing crowdfunding campaigns: pledge, execute-on-success, refund-on-failure | `create_campaign`, `pledge`, `execute_campaign`, `claim_refund` | — |
| `reentrancy` | The storage-flag reentrancy guard used by fund-moving admin/fee functions | `enter`, `exit` | [Reentrancy](../security/reentrancy.md) |
| `search` | Read-only filtered/paginated queries over invoices, merchants, subscriptions, events, and withdrawal proposals | `search_invoices_paginated`, `search_merchants_paginated`, `find_merchant_id` | — |
| `signature_util` | Builds the signed-invoice message and verifies the merchant's ed25519 signature and nonce | `verify_invoice_signature`, `invalidate_nonce` | [Signatures](../security/signatures.md) |
| `stretch_goals` | Funding milestones beyond a campaign's base goal, unlocked once the raise reaches each target | `create_stretch_goal`, `unlock_stretch_goal`, `claim_stretch_goal_reward` | — |
| `subscription` | Recurring billing plans, subscriptions, and interval-gated charging | `create_subscription_plan`, `subscribe`, `charge_subscription`, `cancel_subscription` | [Subscriptions](../concepts/subscriptions.md) |
| `upgrade` | The admin emergency path for replacing the hub's own WASM | `upgrade` | [Upgradeability §the admin path](upgradeability.md#the-admin-path) |
| `vesting` | Admin vesting timeline templates, plus campaign-creator cliff-and-linear fund vesting | `create_creator_vesting`, `release_creator_vesting`, `revoke_creator_vesting` | — |
| `voting` | Dynamic hard-cap voting for adjusting a crowdfund's funding cap mid-campaign | `initiate_hard_cap_voting` | — |
| `kyc` | KYC reviewer roles and per-subject/per-campaign verification request lifecycle | (module-internal; not yet wired into `ShadeTrait`) | — |

Several concept docs are still marked *planned* in [`docs/concepts/README.md`](../concepts/README.md) (invoices/drafts/signed invoices, time-locked fees, escrow and arbiter release, subscriptions and recurring billing, fiat-pegged goals, creator vesting, hard-cap voting, stretch goals, DAO governance, cross-chain bridge, reentrancy guard) — where a module's related doc above links to one of those, follow the "planned" note there rather than expecting a filled-in page yet.

> **Note:** `kyc` is fully implemented (request lifecycle, reviewer roles, per-campaign config) but none of its functions currently appear in `shade_interface.rs`/`shade.rs`. It is reachable only from within the crate today, which is why it has no `ShadeTrait` entry in [`docs/reference/shade-interface.md`](../reference/shade-interface.md).

## What makes a good component boundary

Reading across all 34 modules, the pattern that holds is: **a component owns exactly the storage keys for its feature domain, and every other component reaches that state only through its public functions** — never through a raw `env.storage()` call on a key another module owns. Concretely:

- **Direct function call is the norm.** `invoice::pay_invoice_partial_inner` calls `merchant::get_merchant_account`, `admin::is_accepted_token`, and `platform_fee::route_from_payer` directly — components import each other's public functions like any other Rust module dependency. There is no message-passing or event-driven indirection between components; it's all synchronous, in-process function calls within the same contract invocation.
- **No component reaches into another's storage.** `fiat_goals` needs a campaign's owner and token, so it calls into `campaigns::get_campaign` rather than reading `CampaignKey::Campaign(id)` itself. `stretch_goals` does the same. This is what keeps a component's storage layout an implementation detail: `campaigns` could change how `Campaign` is stored without any other component's code changing, as long as its public functions keep their signatures.
- **Shared cross-cutting concerns go through their own single-purpose component**, not through copy-pasted logic: every fund-moving admin/fee function calls `reentrancy::enter`/`exit`; every merchant-facing mutator calls `pausable_component::assert_not_paused` (via `shade.rs`, not inline); every fee-computing path goes through `platform_fee::compute_split` rather than each caller doing its own basis-point math.
- **Cross-contract calls (to `account`, price oracles, etc.) are declared as a local `#[contractclient]` trait right where they're used** — `invoice.rs` declares `MerchantAccountRefund` and `PriceOracle`, `multisig_withdrawal.rs` declares its own `MerchantAccountWithdraw`, `auto_withdrawal.rs` declares its own `MerchantAccountAutoWithdrawal`. Each of these is a minimal, call-site-scoped trait rather than a shared "external contracts" module — so a component's cross-contract dependency is visible in the same file that uses it. See [Cross-Contract Calls](cross-contract-calls.md) for how these calls behave under Soroban's atomicity and auth-propagation rules.
- **A component that becomes two concerns should split**, and the codebase already shows this: `campaigns` (category/tag `Campaign`) and `campaign` (fee-policy/staking `FeeCampaign`) and `pledge` (all-or-nothing `PledgeCampaign`) all model "a crowdfunding campaign" but are three separate modules with three separate storage keys and record types, precisely because they answered different requirements at different times and merging them would have meant reconciling three incompatible `Campaign`-shaped records under one name. Keep that separation in mind before adding a fourth thing called "campaign."

## How authorization, guards, storage, and events interact with the boundary

These four concerns are handled at different layers, and the layering is what the pattern buys you:

- **Pause guard** — centralized in `shade.rs`. This is the *only* cross-cutting check `shade.rs` applies itself rather than delegating; every other state-changing method's first real line is `pausable_component::assert_not_paused(&env)`, so a reviewer can audit pause coverage by scanning one file instead of 34.
- **Reentrancy guard** — decentralized, opt-in per function. `reentrancy::enter`/`exit` is called from inside the handful of functions that make untrusted external calls after a state read but need to prevent being re-entered before they finish (`admin.rs`'s token/fee functions, `platform_fee.rs`'s fee-config setters, `kyc.rs`'s mutators, `campaign.rs`'s `create_campaign`). It is not applied uniformly — `invoice::pay_invoice` does not call `reentrancy::enter`, for instance — so don't assume every fund-moving path is guarded; check the specific component.
- **Feature-specific authorization** — always inside the component, always as close to the top of the function as possible, using `require_auth()` plus (where relevant) an ownership or role check against stored state (`core::assert_admin`, `merchant::get_merchant_id(...) == invoice.merchant_id`, `access_control::has_role`).
- **Storage ownership** — one component per key-enum-and-domain; see the [Storage and layering](#storage-and-layering) note below and [`contracts/shade/src/types.rs`](../../contracts/shade/src/types.rs)'s module doc comment for the full list of key enums.
- **Event emission** — always inside the component, at the point where the state change it describes has just been committed, using the helpers in [`contracts/shade/src/events.rs`](../../contracts/shade/src/events.rs). See [Events reference](../reference/events.md) for the full catalog and topic conventions.

## Storage and layering

- **`types.rs`** defines every stored record and every storage-key enum (`DataKey`, `EventKey`, `CampaignKey`, `BackerKey`, `StretchKey`, `VestingKey`, `FiatGoalKey`, `AnalyticsKey`, `NftKey`, `GovKey`, `BridgeKey`, `MultiSigKey`) — split across multiple enums because Soroban caps every `#[contracttype]` enum at 50 cases. A component's storage keys live in exactly the enum documented for its domain in that file's module comment.
- **`errors.rs`** likewise splits error codes into domain enums (`ContractError`, `GovernanceError`, `EventError`, `MultiSigError`, `EscrowError`, `CampaignError`, `StretchGoalError`, `VestingError`, `FiatGoalError`, `AnalyticsError`) with non-overlapping numeric ranges, so a raw error code observed on-chain maps back to exactly one variant regardless of which enum produced it.
- **`events.rs`** holds every `#[contractevent]` struct and its `publish_*` helper. A component never constructs an event struct outside this file's helpers.
- **`components/`** holds the logic, one module per feature, per the inventory above.
- **`shade_interface.rs`** and **`shade.rs`** are the two files described in [The three-file pattern](#the-three-file-pattern) — the public surface and its wiring.

For the workspace-level view of how these files relate to the rest of the contracts (not just `shade`), see [Architecture overview](overview.md).

## Related pages

- [Architecture overview](overview.md)
- [Cross-Contract Calls and the Factory Pattern](cross-contract-calls.md)
- [Upgradeability, WASM hash management, and versioning](upgradeability.md)
- [Workspace and Crate Layout](workspace-layout.md)
- [`ShadeTrait` function reference](../reference/shade-interface.md)
- [Events reference](../reference/events.md)

← [Back to Architecture](README.md)
