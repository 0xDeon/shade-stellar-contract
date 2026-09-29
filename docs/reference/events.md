# Events reference

Every event emitted anywhere in this workspace — the `shade` hub contract and all eight satellite contracts — with its topic, data payload, emitting function, and the state transition it marks. Built by reading every `#[contractevent]` struct and every `publish_*`/`.publish(env)` call site in the repository, not just `contracts/shade/src/events.rs`.

## How Soroban events work here (read before the tables)

Every event in this workspace uses the SDK's `#[contractevent]` derive macro on a plain struct, then calls `.publish(env)` on an instance of it (usually through a `publish_*` helper function). Two conventions appear:

- **Default (used almost everywhere):** `#[contractevent]` with no arguments. The macro derives the event's topic from the **struct name in snake_case** — e.g. `InvoicePaidEvent` publishes under the topic `invoice_paid_event`. [`contracts/shade/src/events.rs`](../../contracts/shade/src/events.rs)'s own module doc comment states this explicitly and warns that Soroban caps topic Symbols at 32 characters, which is why struct names in that file are kept short.
- **Explicit topic override (used only in `account`):** `#[contractevent(topics = ["account_restricted"])]` on `AccountRestrictedEvent` sets the topic string directly rather than deriving it from the struct name.
- **`data_format` override:** `account`'s `AccountVerifiedEvent` uses `#[contractevent(data_format = "single-value")]`, publishing its one field (`timestamp`) as a bare value instead of the default field-map encoding.

**None of these events declare a multi-field topic tuple.** Every event in this workspace publishes under a single topic (the derived or overridden Symbol above); the event's *data* — every field on the struct — is what carries the rich, multi-field payload, encoded by default as a `Map<Symbol, Val>` keyed by field name (this is what lets tests decode a field by name, as shown in [Decoding an event](#decoding-a-real-event) below). "Filter by topic" in this workspace therefore means "filter by event name / contract address," not "filter by a structured topic tuple" — see [Filter examples](#filter-examples) for what this means in practice.

## Index by domain

- [Core payments (shade)](#core-payments-shade) — admin, tokens, fees, merchants, invoices, payments
- [Access control & pause (shade)](#access-control--pause-shade)
- [Subscriptions (shade)](#subscriptions-shade)
- [Bridge & cross-chain pledges (shade)](#bridge--cross-chain-pledges-shade)
- [DAO governance & upgrades (shade)](#dao-governance--upgrades-shade)
- [Event ticketing (shade)](#event-ticketing-shade)
- [Escrow (shade)](#escrow-shade)
- [NFT rewards (shade)](#nft-rewards-shade)
- [Campaigns, categories & tags (shade)](#campaigns-categories--tags-shade)
- [Backer rewards & crowdfund comments (shade)](#backer-rewards--crowdfund-comments-shade)
- [Pledge campaigns & leaderboard (shade)](#pledge-campaigns--leaderboard-shade)
- [Fee-policy / staking / affiliate campaigns (shade)](#fee-policy--staking--affiliate-campaigns-shade)
- [Stretch goals, vesting & fiat goals (shade)](#stretch-goals-vesting--fiat-goals-shade)
- [Multi-sig withdrawals & search telemetry (shade)](#multi-sig-withdrawals--search-telemetry-shade)
- [Campaign analytics exports (shade)](#campaign-analytics-exports-shade)
- [`account` contract](#account-contract)
- [`escrow` contract](#escrow-contract)
- [`ticketing` contract](#ticketing-contract)
- [`ticketing_factory` contract](#ticketing_factory-contract)
- [`subscription` contract](#subscription-contract)
- [`crowdfund` contract](#crowdfund-contract)
- [`crowdfund_factory` contract](#crowdfund_factory-contract)

> **Note:** `escrow_factory` and `crowdfund_factory`'s deploy-time housekeeping aside, the standalone contracts each define their own event set independent of `shade`'s — there is no shared event vocabulary across contracts, even where the domain overlaps (e.g. `shade`'s `EscrowCreatedEvent` vs. the standalone `escrow` contract's `ReleasedEvent`).

## Core payments (shade)

All defined and emitted from [`contracts/shade/src/events.rs`](../../contracts/shade/src/events.rs) and [`contracts/shade/src/components/`](../../contracts/shade/src/components/).

| Event | Data fields | Emitting function | Emission point | State transition |
|---|---|---|---|---|
| `InitalizedEvent` *(sic)* | `admin: Address`, `timestamp: u64` | `shade.rs::initialize` | After writing `DataKey::Admin`/`PlatformAccount`/`ContractInfo` | Contract initialized |
| `TokenAddedEvent` | `token: Address`, `timestamp: u64` | `admin::add_accepted_token`/`add_accepted_tokens` | After appending to `DataKey::AcceptedTokens`, inside the reentrancy guard | Token added to the global accepted-token whitelist |
| `TokenRemovedEvent` | `token: Address`, `timestamp: u64` | `admin::remove_accepted_token` | After rewriting `DataKey::AcceptedTokens` without the token | Token removed from the whitelist |
| `MerchantRegisteredEvent` | `merchant: Address`, `merchant_id: u64`, `timestamp: u64` | `merchant::register_merchant` | After writing `Merchant`/`MerchantId`/`MerchantCount` | New merchant registered |
| `MerchantAccountDeployedEvent` | `merchant: Address`, `contract: Address`, `timestamp: u64` | (declared; see note) | — | Merchant `account` contract deployed via `account_factory` |
| `MerchantStatusChangedEvent` | `merchant_id: u64`, `active: bool`, `timestamp: u64` | `merchant::set_merchant_status` | After writing the updated `Merchant.active` | Merchant activated/deactivated |
| `InvoiceCreatedEvent` | `invoice_id: u64`, `merchant: Address`, `amount: i128`, `token: Address` | `invoice::create_invoice`, `create_fiat_invoice`, `create_invoice_signed`, `finalize_invoice` | After writing the new/finalized `Invoice` record | Invoice created (or a draft finalized into `Pending`) |
| `InvoiceRefundedEvent` | `invoice_id: u64`, `merchant: Address`, `amount: i128`, `timestamp: u64` | `invoice::refund_invoice`, `refund_invoice_partial` (full-refund branch) | After the refund token transfer and `Invoice.status = Refunded` write | Invoice fully refunded |
| `InvoicePartiallyRefundedEvent` | `invoice_id: u64`, `merchant: Address`, `amount: i128`, `total_amount_refunded: i128`, `timestamp: u64` | `invoice::refund_invoice_partial` (partial branch) | Same point, when `total_refund < invoice.amount` | Invoice partially refunded |
| `MerchantVerifiedEvent` | `merchant_id: u64`, `status: bool`, `timestamp: u64` | `merchant::verify_merchant` | After writing `Merchant.verified` | Merchant verification status changed |
| `MerchantWebhookSetEvent` | `merchant: Address`, `merchant_id: u64`, `webhook: String`, `timestamp: u64` | `merchant::set_merchant_webhook` | After writing `Merchant.webhook` | Merchant webhook URL updated |
| `MerchantKeySetEvent` | `merchant: Address`, `key: BytesN<32>`, `timestamp: u64` | `merchant::set_merchant_key` | After writing `DataKey::MerchantKey` | Merchant's signed-invoice verification key set |
| `RoleGrantedEvent` | `admin: Address`, `user: Address`, `role: Role`, `timestamp: u64` | `access_control::grant_role` | After writing `DataKey::Role` | Role granted to an address |
| `RoleRevokedEvent` | `admin: Address`, `user: Address`, `role: Role`, `timestamp: u64` | `access_control::revoke_role` | After removing `DataKey::Role` | Role revoked |
| `ContractPausedEvent` | `admin: Address`, `timestamp: u64` | `pausable::pause` | After writing `DataKey::Paused = true` | Contract paused |
| `ContractUnpausedEvent` | `admin: Address`, `timestamp: u64` | `pausable::unpause` | After writing `DataKey::Paused = false` | Contract unpaused |
| `FeeProposedEvent` | `admin: Address`, `token: Address`, `fee: i128`, `timestamp: u64` | `admin::propose_fee` | After writing `DataKey::PendingTokenFee` | Time-locked fee change proposed |
| `FeeSetEvent` | `admin: Address`, `token: Address`, `fee: i128`, `timestamp: u64` | `admin::set_fee`, `execute_fee` | After writing `DataKey::TokenFee` | Fee set (directly, or via executing a pending proposal) |
| `PlatformAccountSetEvent` | `admin: Address`, `account: Address`, `timestamp: u64` | `admin::set_platform_account` | After writing `DataKey::PlatformAccount` | Platform fee-recipient account changed |
| `TokenOracleSetEvent` | `admin: Address`, `token: Address`, `oracle: Address`, `timestamp: u64` | `admin::set_token_oracle` | After writing `DataKey::TokenOracle` | Price oracle configured for a token |
| `ContractUpgradedEvent` | `new_wasm_hash: BytesN<32>`, `timestamp: u64` | `upgrade::upgrade`, `governance::finalize_upgrade` (on success) | After `env.deployer().update_current_contract_wasm(...)` | Hub contract's own WASM replaced |
| `AccountRestrictedEvent` | `merchant: Address`, `status: bool`, `caller: Address`, `timestamp: u64` | `merchant::restrict_merchant_account` | After the cross-contract `restrict_account` call on the merchant's `account` contract | Merchant account restricted/unrestricted |
| `FeeDiscountAppliedEvent` | `merchant: Address`, `volume: i128`, `discount_bps: i128`, `timestamp: u64` | (declared; volume-discount path in `platform_fee::apply_volume_discount`'s callers) | — | Volume-based fee discount applied |
| `InvoicePaidEvent` | `invoice_id: u64`, `merchant_id: u64`, `merchant_account: Address`, `payer: Address`, `amount: i128`, `fee: i128`, `merchant_amount: i128`, `token: Address`, `timestamp: u64` | `invoice::pay_invoice_partial_inner` | After the `Invoice` write (status now `Paid` or `PartiallyPaid`) | A payment (full or partial) landed on an invoice |
| `FiatInvoicePricedEvent` | `invoice_id: u64`, `token: Address`, `resolved_amount: i128`, `timestamp: u64` | `invoice::create_fiat_invoice`, `refresh_fiat_invoice_quote` | Immediately after resolving the invoice's crypto amount from the fiat/oracle price | A fiat-priced invoice's crypto amount was (re-)resolved |
| `PaymentSplitRoutedEvent` | `invoice_id: u64`, `merchant_account: Address`, `platform_account: Address`, `merchant_amount: i128`, `platform_amount: i128`, `token: Address`, `timestamp: u64` | `invoice::pay_invoice_partial_inner` | Right after `InvoicePaidEvent`, same call | Fee/merchant split for an invoice payment was routed |
| `PlatformFeeRoutedEvent` | `route_kind: u32`, `ref_id: u64`, `merchant_id: u64`, `merchant: Address`, `merchant_account: Address`, `platform_account: Address`, `payer: Address`, `gross_amount: i128`, `platform_fee: i128`, `merchant_amount: i128`, `token: Address`, `fee_bps_applied: i128`, `timestamp: u64` | `platform_fee::finalize_route` (called from `route_from_payer`/`route_from_allowance`) | Right after the split token transfers execute, before the caller's own event | Any fee-routed payment (invoice, subscription charge, ticket purchase) settled |
| `MerchantPlatformFeeSetEvent` | `caller: Address`, `merchant_id: u64`, `token: Address`, `fee_bps: i128`, `timestamp: u64` | `platform_fee::set_merchant_platform_fee` | After writing `DataKey::MerchantPlatformFee` | Per-merchant fee override set |
| `PlatformFeeClearedEvent` | `caller: Address`, `merchant_id: u64`, `token: Address`, `timestamp: u64` | `platform_fee::clear_merchant_platform_fee` | After removing `DataKey::MerchantPlatformFee` | Per-merchant fee override cleared |
| `InvoiceCancelledEvent` | `invoice_id: u64`, `merchant: Address`, `timestamp: u64` | `invoice::void_invoice` | After `Invoice.status = Cancelled` | Pending invoice voided |
| `InvoiceAmendedEvent` | `invoice_id: u64`, `merchant: Address`, `old_amount: i128`, `new_amount: i128`, `timestamp: u64` | `invoice::amend_invoice` | After writing the amended `Invoice` | Pending invoice's amount/description edited |
| `NonceInvalidatedEvent` | `merchant: Address`, `nonce: BytesN<32>`, `timestamp: u64` | `signature_util::invalidate_nonce` | After writing `DataKey::UsedNonce` | A signed-invoice nonce was consumed |
| `AdminTransferProposedEvent` | `current_admin: Address`, `proposed_admin: Address`, `timestamp: u64` | `admin::propose_admin_transfer` | After writing `DataKey::PendingAdmin` | Admin handover proposed |
| `AdminTransferAcceptedEvent` | `old_admin: Address`, `new_admin: Address`, `timestamp: u64` | `admin::accept_admin_transfer` | After writing `DataKey::Admin` | Admin handover completed |
| `AccountWasmHashSetEvent` | `admin: Address`, `wasm_hash: BytesN<32>`, `timestamp: u64` | `admin::set_account_wasm_hash` | After writing `DataKey::AccountWasmHash` | WASM hash for future merchant-account deployments updated |
| `MerchantTokensSetEvent` | `merchant: Address`, `tokens: Vec<Address>`, `timestamp: u64` | `merchant::set_merchant_accepted_tokens` | After writing `DataKey::MerchantTokens` | Merchant's own accepted-token list set |
| `MerchantTokenRemovedEvent` | `merchant: Address`, `token: Address`, `timestamp: u64` | `merchant::remove_merchant_accepted_token` | After rewriting `DataKey::MerchantTokens` | One token removed from a merchant's accepted list |
| `AutoWithdrawThresholdEvent` | `merchant_id: u64`, `token: Address`, `threshold: i128` | `auto_withdrawal::set_auto_withdrawal_threshold` | After writing the merchant's threshold list | Auto-withdrawal threshold set |
| `AutoWithdrawRecipientEvent` | `merchant_id: u64`, `recipient: Address` | `auto_withdrawal::set_auto_withdrawal_recipient` | After writing `Merchant.auto_withdrawal_recipient` | Auto-withdrawal recipient set |
| `AutoWithdrawalTriggeredEvent` | `merchant_id: u64`, `token: Address`, `amount: i128`, `recipient: Address` | `auto_withdrawal::trigger_auto_withdrawal` | After the cross-contract `withdraw_to` call on the merchant's `account` contract | Balance crossed the threshold; funds swept |
| `EscrowExpiredRefundEvent` | `invoice_id: u64`, `buyer: Address`, `amount: i128`, `token: Address`, `timestamp: u64` | `invoice::claim_refund` | After the refund transfer and `Invoice.status = Refunded` write | Expired unfulfilled invoice refunded to the buyer |

## Access control & pause (shade)

Covered above under core payments (`RoleGrantedEvent`, `RoleRevokedEvent`, `ContractPausedEvent`, `ContractUnpausedEvent`) — listed there because they live in the same `events.rs` section and share the same emission points.

## Subscriptions (shade)

| Event | Data fields | Emitting function | Emission point | State transition |
|---|---|---|---|---|
| `SubscriptionPlanCreatedEvent` | `plan_id: u64`, `merchant: Address`, `token: Address`, `amount: i128`, `interval: u64`, `timestamp: u64` | `subscription::create_subscription_plan` | After writing the new `SubscriptionPlan` | Plan created |
| `SubscribedEvent` | `subscription_id: u64`, `plan_id: u64`, `customer: Address`, `timestamp: u64` | `subscription::subscribe` | After writing the new `Subscription` | Customer enrolled in a plan |
| `SubscriptionChargedEvent` | `subscription_id: u64`, `plan_id: u64`, `customer: Address`, `merchant: Address`, `amount: i128`, `fee: i128`, `token: Address`, `timestamp: u64` | `subscription::charge_subscription` | After the fee-routed charge transfer and `Subscription.last_charged` write | A billing interval was charged |
| `SubscriptionCancelledEvent` | `subscription_id: u64`, `caller: Address`, `timestamp: u64` | `subscription::cancel_subscription` | After `Subscription.status = Cancelled` | Subscription cancelled (by customer or merchant) |
| `PlanDeactivatedEvent` | `plan_id: u64`, `merchant: Address`, `timestamp: u64` | `subscription::deactivate_plan` | After `SubscriptionPlan.active = false` | Plan closed to new subscribers |
| `PlanSearchExecutedEvent` | `caller: Address`, `result_count: u32`, `timestamp: u64` | `search::search_subscription_plans` | After building the result `Vec` | Telemetry: a plan search query ran |

## Bridge & cross-chain pledges (shade)

| Event | Data fields | Emitting function | Emission point | State transition |
|---|---|---|---|---|
| `BridgePlaceholderEvent` | `caller: Address`, `payload: CrossChainBridgePayload`, `timestamp: u64` | `shade.rs::emit_bridge_placeholder` | Directly, no state write — this call moves no funds | Off-chain bridge-integration intent signaled (see [Payment payloads, swap routing, and the cross-chain bridge placeholder](../concepts/payment-payloads-and-routing.md)) |
| `BridgeListenerRegisteredEvent` | `admin: Address`, `listener: Address`, `timestamp: u64` | `bridge::register_bridge_listener` | After writing `BridgeKey::BridgeListener` | Relayer authorized |
| `BridgeListenerRemovedEvent` | `admin: Address`, `listener: Address`, `timestamp: u64` | `bridge::remove_bridge_listener` | After removing `BridgeKey::BridgeListener` | Relayer authorization revoked |
| `BridgeDepositRecordedEvent` | `deposit_id: u64`, `listener: Address`, `source_chain: String`, `source_tx_id: BytesN<32>`, `token: Address`, `amount: i128`, `recipient: Address`, `timestamp: u64` | `bridge::record_bridge_deposit` | After writing `BridgeKey::BridgeDeposit` and the idempotency marker | Confirmed external-chain deposit credited |
| `CrossChainPledgeCreatedEvent` | `pledge_id: u64`, `source_chain: String`, `source_pledge_id: u64`, `merchant: Address`, `payer: Address`, `token: Address`, `amount: i128`, `timestamp: u64` | `cross_chain_pledge::create_pledge` (via `publish_cross_chain_pledge_created`) | After writing the new `CrossChainPledge` | Inbound cross-chain pledge recorded |
| `CrossChainPledgeUpdatedEvent` | `pledge_id: u64`, `status: CrossChainPledgeStatus`, `timestamp: u64` | `cross_chain_pledge` status-update path (via `publish_cross_chain_pledge_updated`) | After writing the updated `CrossChainPledge.status` | Cross-chain pledge status changed |

## DAO governance & upgrades (shade)

| Event | Data fields | Emitting function | Emission point | State transition |
|---|---|---|---|---|
| `GovMemberAddedEvent` | `admin: Address`, `member: Address`, `member_count: u32`, `timestamp: u64` | `governance::add_gov_member` | After writing `GovKey::Member` and incrementing the count | Council member added |
| `GovMemberRemovedEvent` | `admin: Address`, `member: Address`, `member_count: u32`, `timestamp: u64` | `governance::remove_gov_member` | After removing `GovKey::Member` | Council member removed |
| `GovConfigSetEvent` | `admin: Address`, `voting_period: u64`, `quorum_bps: u32`, `timestamp: u64` | `governance::set_governance_config` | After writing `GovState` | Voting window/quorum configured |
| `UpgradeProposedEvent` | `proposal_id: u64`, `proposer: Address`, `wasm_hash: BytesN<32>`, `voting_ends_at: u64`, `timestamp: u64` | `governance::propose_upgrade` | After writing the new `UpgradeProposal` | Upgrade proposal opened |
| `UpgradeVoteCastEvent` | `proposal_id: u64`, `voter: Address`, `approve: bool`, `approvals: u32`, `rejections: u32`, `timestamp: u64` | `governance::vote_on_upgrade` | After writing the vote and updated tallies | A council member voted |
| `UpgradeProposalFinalizedEvent` | `proposal_id: u64`, `executor: Address`, `approved: bool`, `approvals: u32`, `rejections: u32`, `member_count: u32`, `timestamp: u64` | `governance::finalize_upgrade` | After the proposal is marked `Executed`/`Defeated` (and, if approved, right before/alongside `ContractUpgradedEvent`) | Voting window closed and the proposal resolved |

## Event ticketing (shade)

| Event | Data fields | Emitting function | Emission point | State transition |
|---|---|---|---|---|
| `EventCreatedEvent` | `event_id: u64`, `merchant: Address`, `merchant_id: u64`, `name: String`, `ticket_price: i128`, `token: Address`, `capacity: u32`, `event_date: u64`, `royalty_bps: u32`, `timestamp: u64` | `event::create_event` | After writing the new `Event` | Ticketed event created |
| `TicketPurchasedEvent` | `ticket_id: u64`, `event_id: u64`, `merchant_id: u64`, `buyer: Address`, `amount: i128`, `fee: i128`, `merchant_amount: i128`, `token: Address`, `timestamp: u64` | `event::purchase_ticket`, `purchase_tickets_bulk` | After the fee-routed purchase transfer and `Ticket`/`Event.sold` writes | Ticket(s) sold |
| `TicketResoldEvent` | `ticket_id: u64`, `event_id: u64`, `merchant_id: u64`, `seller: Address`, `buyer: Address`, `resale_price: i128`, `royalty: i128`, `seller_proceeds: i128`, `token: Address`, `timestamp: u64` | `event::resell_ticket` | After the resale transfer (royalty to organizer, remainder to seller) | Ticket resold on the secondary market |
| `TicketListedEvent` | `ticket_id: u64`, `seller: Address`, `price: i128`, `timestamp: u64` | `event`'s listing path | After writing `EventKey::TicketListing` | Ticket listed for resale |
| `TicketListingCancelledEvent` | `ticket_id: u64`, `seller: Address`, `timestamp: u64` | `event`'s listing path | After removing `EventKey::TicketListing` | Resale listing cancelled |
| `TicketListingSoldEvent` | `ticket_id: u64`, `seller: Address`, `buyer: Address`, `resale_price: i128`, `royalty: i128`, `timestamp: u64` | `event`'s listing path | After the listed-resale transfer | Listed ticket sold |
| `EventSearchExecutedEvent` | `caller: Address`, `result_count: u32`, `timestamp: u64` | `search::search_events` | After building the result `Vec` | Telemetry: an event search query ran |

## Escrow (shade)

| Event | Data fields | Emitting function | Emission point | State transition |
|---|---|---|---|---|
| `EscrowCreatedEvent` | `id: u64`, `seller: Address`, `buyer: Address`, `token: Address`, `amount: i128`, `invoice_id: Option<u64>`, `timestamp: u64` | `escrow::create_escrow` | After writing the new `Escrow` (status `Created`) | Escrow opened |
| `EscrowFundedEvent` | `escrow_id: u64`, `buyer: Address`, `seller: Address`, `token: Address`, `amount: i128`, `timestamp: u64` | `escrow::fund_escrow` | After the buyer's deposit transfer and `Escrow.status = Funded` write | Escrow funded |
| `EscrowReleasedEvent` | `escrow_id: u64`, `buyer: Address`, `seller: Address`, `token: Address`, `merchant_amount: i128`, `fee: i128`, `timestamp: u64` | `escrow::release_escrow` | After the release transfer and `Escrow.status = Released` write | Escrow released to the seller |
| `EscrowRefundedEvent` | `escrow_id: u64`, `seller: Address`, `buyer: Address`, `token: Address`, `amount: i128`, `timestamp: u64` | `escrow::refund_escrow` | After the refund transfer and `Escrow.status = Refunded` write | Escrow refunded to the buyer |

## NFT rewards (shade)

| Event | Data fields | Emitting function | Emission point | State transition |
|---|---|---|---|---|
| `NftCollectionCreatedEvent` | `id: u64`, `merchant_id: u64`, `merchant_addr: Address`, `name: String`, `base_uri: String`, `max_supply: u64`, `royalty_bps: u32`, `timestamp: u64` | `nft::create_nft_collection` | After writing the new `NftCollection` | Collection created |
| `NftMintedEvent` | `nft_id: u64`, `collection_id: u64`, `merchant_id: u64`, `recipient: Address`, `token_uri: String`, `timestamp: u64` | `nft::mint_nft` | After writing the new `Nft` and incrementing `minted` | NFT minted |
| `NftBatchMintedEvent` | `collection_id: u64`, `merchant_id: u64`, `count: u32`, `timestamp: u64` | `nft::batch_mint_nfts` | After minting the whole batch | Multiple NFTs minted in one call |
| `NftTransferredEvent` | `nft_id: u64`, `collection_id: u64`, `from: Address`, `to: Address`, `timestamp: u64` | `nft::transfer_nft` | After writing `Nft.owner` | NFT ownership transferred |
| `NftBurnedEvent` | `nft_id: u64`, `collection_id: u64`, `owner: Address`, `timestamp: u64` | `nft::burn_nft` | After `Nft.status = Burned` | NFT burned |
| `NftRewardClaimedEvent` | `nft_id: u64`, `collection_id: u64`, `claimer: Address`, `timestamp: u64` | `nft::claim_nft_reward` | After marking the reward claimed | Reward NFT claimed by its recipient |
| `NftCollectionDeactivatedEvent` | `collection_id: u64`, `merchant_addr: Address`, `timestamp: u64` | `nft::deactivate_nft_collection` | After `NftCollection.active = false` | Collection closed to further minting |

## Campaigns, categories & tags (shade)

| Event | Data fields | Emitting function | Emission point | State transition |
|---|---|---|---|---|
| `CampaignCategoryCreatedEvent` | `category_id: u64`, `admin: Address`, `name: String`, `description: String`, `timestamp: u64` | `campaigns::create_category` | After writing the new `CampaignCategory` | Category created |
| `CampaignCategoryUpdatedEvent` | `category_id: u64`, `admin: Address`, `name: String`, `description: String`, `active: bool`, `timestamp: u64` | `campaigns::update_category` | After writing the updated category | Category edited |
| `CampaignTagCreatedEvent` | `tag_id: u64`, `creator: Address`, `name: String`, `timestamp: u64` | `campaigns::create_tag` | After writing the new `CampaignTag` | Tag created |
| `CampaignCreatedEvent` | `campaign_id: u64`, `merchant: Address`, `merchant_id: u64`, `title: String`, `description: String`, `category_id: u64`, `tags: Vec<u64>`, `goal_amount: i128`, `token: Address`, `deadline: u64`, `timestamp: u64` | `campaigns::create_campaign` | After writing the new `Campaign` | Campaign published |
| `CampaignUpdatedEvent` | `campaign_id: u64`, `merchant: Address`, `title: String`, `description: String`, `timestamp: u64` | `campaigns::update_campaign` | After writing the updated `Campaign` | Title/description edited |
| `CampaignStatusChangedEvent` | `campaign_id: u64`, `merchant: Address`, `active: bool`, `timestamp: u64` | `campaigns::set_campaign_active` | After writing `Campaign.active` | Campaign activated/deactivated |
| `CampaignTagAddedEvent` | `campaign_id: u64`, `merchant: Address`, `tag_id: u64`, `timestamp: u64` | `campaigns::add_campaign_tag` | After writing the campaign's tag list | Tag attached |
| `CampaignTagRemovedEvent` | `campaign_id: u64`, `merchant: Address`, `tag_id: u64`, `timestamp: u64` | `campaigns::remove_campaign_tag` | After rewriting the campaign's tag list | Tag detached |
| `CampaignContributionEvent` | `campaign_id: u64`, `contributor: Address`, `amount: i128`, `raised_amount: i128`, `goal_amount: i128`, `timestamp: u64` | `campaigns::record_contribution` | After writing `Campaign.raised_amount` | A contribution recorded against a category/tag campaign |

## Backer rewards & crowdfund comments (shade)

| Event | Data fields | Emitting function | Emission point | State transition |
|---|---|---|---|---|
| `BackerCampaignCreatedEvent` | `campaign_id: u64`, `merchant_addr: Address`, `merchant_id: u64`, `name: String`, `token: Address`, `deadline: u64`, `timestamp: u64` | `backer_rewards::create_backer_campaign` | After writing the new `BackerCampaign` | Backer-reward campaign created |
| `BackerRewardTiersSetEvent` | `campaign_id: u64`, `merchant_addr: Address`, `tiers_len: u32`, `timestamp: u64` | `backer_rewards::set_backer_reward_tiers` | After writing `BackerKey::RewardTiers` | Reward tier ladder published |
| `BackerPledgeRecordedEvent` | `campaign_id: u64`, `backer: Address`, `amount: i128`, `new_pledge: i128`, `timestamp: u64` | `backer_rewards::pledge_to_campaign` | After writing the backer's cumulative pledge | Pledge recorded |
| `BackerTierSelectedEvent` | `campaign_id: u64`, `backer: Address`, `tier_index: u32`, `min_pledge: i128`, `tier_len: u32`, `timestamp: u64` | `backer_rewards::select_backer_reward_tier` | After writing `BackerKey::SelectedTier` | Backer selected a reward tier |
| `BackerRewardFulfilledEvent` | `campaign_id: u64`, `merchant_addr: Address`, `backer: Address`, `tier_index: u32`, `pledge: i128`, `timestamp: u64` | `backer_rewards::fulfill_backer_reward` | After writing `BackerKey::RewardFulfilled` | Merchant marked a backer's reward fulfilled |
| `BackerPerkClaimedEvent` | `campaign_id: u64`, `backer: Address`, `tier_index: u32`, `perk_index: u32`, `name: String`, `timestamp: u64` | `backer_rewards::claim_backer_perk` | After writing `BackerKey::PerkClaimed` | Backer claimed one perk |
| `BackerCommentCreatedEvent` | `comment_id: u64`, `crowdfund_id: u64`, `author: Address`, `content_len: u64`, `now: u64` | `comments::create_comment` | After writing the new `BackerComment` | Comment posted |
| `BackerCommentFlaggedEvent` | `comment_id: u64`, `flagger: Address`, `reason_len: u64`, `flag_count: u32`, `now: u64` | `comments`'s flag path | After writing the `CommentFlag` and incrementing `flag_count` | Comment flagged |
| `BackerCommentRemovedEvent` | `comment_id: u64`, `crowdfund_id: u64`, `moderator: Address`, `now: u64` | `comments`'s moderation path | After `BackerComment.status = Removed` | Comment removed by a moderator |

## Pledge campaigns & leaderboard (shade)

| Event | Data fields | Emitting function | Emission point | State transition |
|---|---|---|---|---|
| `PledgeCampaignCreatedEvent` | `campaign_id: u64`, `merchant: Address`, `merchant_id: u64`, `title: String`, `goal: i128`, `token: Address`, `deadline: u64`, `timestamp: u64` | `pledge::create_campaign` | After writing the new `PledgeCampaign` | All-or-nothing campaign created |
| `PledgeMadeEvent` | `pledge_id: u64`, `campaign_id: u64`, `contributor: Address`, `amount: i128`, `token: Address`, `timestamp: u64` | `pledge::pledge` | After the pledge transfer and writing the new `Pledge` | Contribution pledged |
| `CampaignExecutedEvent` | `campaign_id: u64`, `merchant_address: Address`, `raised: i128`, `timestamp: u64` | `pledge::execute_campaign` | After `PledgeCampaign.status = Executed` and the payout transfer to the merchant | Campaign reached its goal and funds released to the merchant |
| `CampaignCancelledEvent` | `campaign_id: u64`, `merchant_address: Address`, `timestamp: u64` | `pledge::cancel_campaign` | After `PledgeCampaign.status = Cancelled` | Campaign cancelled before its deadline |
| `PledgeRefundedEvent` | `pledge_id: u64`, `campaign_id: u64`, `contributor: Address`, `amount: i128`, `token: Address`, `timestamp: u64` | `pledge::claim_refund` | After the refund transfer and `Pledge.status = Refunded` | One contributor refunded |
| `CampaignBatchRefundedEvent` | `campaign_id: u64`, `total_refunded: i128`, `count: u32`, `timestamp: u64` | `pledge::batch_refund` | After refunding every outstanding pledge on a failed campaign | Batch refund completed for a whole campaign |
| `LeaderboardUpdatedEvent` | `campaign_id: u64`, `donor: Address`, `amount: i128`, `new_total: i128`, `timestamp: u64` | `leaderboard::track_donation` | After updating the top-donors list | Donor leaderboard updated |

## Fee-policy / staking / affiliate campaigns (shade)

| Event | Data fields | Emitting function | Emission point | State transition |
|---|---|---|---|---|
| `FeeCampaignCreatedEvent` | `campaign_id: u64`, `owner: Address`, `name: String`, `charity: bool`, `fee_waiver_bps: u32`, `discount_bps: u32`, `timestamp: u64` | `campaign::create_campaign` | After writing the new `FeeCampaign`, inside the reentrancy guard | Fee-policy campaign created |
| `CampaignFeePolicySetEvent` | `campaign_id: u64`, `caller: Address`, `fee_waiver_bps: u32`, `discount_bps: u32`, `timestamp: u64` | `campaign::configure_campaign_fee_policy` | After writing the updated fee policy | Fee waiver/discount policy changed |
| `CampaignContribRecordedEvent` | `campaign_id: u64`, `caller: Address`, `amount: i128`, `total_raised: i128`, `timestamp: u64` | `campaign::record_campaign_contribution` | After writing `FeeCampaign.total_raised` | Contribution recorded against a fee-policy campaign |
| `CampaignStakedEvent` | `campaign_id: u64`, `caller: Address`, `amount: i128`, `staked: i128`, `timestamp: u64` | `campaign::stake_campaign` | After the stake transfer and writing `CampaignParticipant.staked` | Participant staked into a campaign |
| `CampaignSlashedEvent` | `campaign_id: u64`, `participant_address: Address`, `amount: i128`, `staked: i128`, `timestamp: u64` | `campaign::slash_campaign_stake` | After writing the reduced stake and `total_slashed` | A participant's stake was slashed |
| `AffiliateRegisteredEvent` | `campaign_id: u64`, `affiliate_address: Address`, `commission_bps: u32`, `timestamp: u64` | `campaign::register_affiliate` | After writing the new `CampaignAffiliate` | Affiliate registered on a campaign |
| `AffiliateCommissionPaidEvent` | `campaign_id: u64`, `affiliate_address: Address`, `amount: i128`, `total_paid: i128`, `timestamp: u64` | `campaign::pay_affiliate_commission` | After the commission transfer | Affiliate commission paid out |

## Stretch goals, vesting & fiat goals (shade)

| Event | Data fields | Emitting function | Emission point | State transition |
|---|---|---|---|---|
| `StretchGoalCreatedEvent` | `goal_id: u64`, `campaign_id: u64`, `merchant: Address`, `target_amount: i128`, `base_goal_amount: i128`, `goal_count: u32`, `timestamp: u64` | `stretch_goals::create_stretch_goal` | After writing the new `StretchGoal` | Milestone defined |
| `StretchGoalUnlockedEvent` | `goal_id: u64`, `campaign_id: u64`, `merchant: Address`, `target_amount: i128`, `raised_amount: i128`, `timestamp: u64` | `stretch_goals::unlock_stretch_goal` | After `StretchGoal.status = Unlocked` | Campaign's raise reached the goal's target |
| `StretchGoalCancelledEvent` | `goal_id: u64`, `campaign_id: u64`, `merchant: Address`, `timestamp: u64` | `stretch_goals::cancel_stretch_goal` | After `StretchGoal.status = Cancelled` | Un-unlocked goal retired |
| `StretchRewardGrantedEvent` | `goal_id: u64`, `campaign_id: u64`, `backer: Address`, `reward_amount: i128`, `reward_count: u32`, `total_reward_amount: i128`, `timestamp: u64` | `stretch_goals::grant_stretch_goal_reward` | After writing the new `StretchGoalReward` | Reward granted to a backer |
| `StretchRewardClaimedEvent` | `goal_id: u64`, `campaign_id: u64`, `backer: Address`, `reward_amount: i128`, `timestamp: u64` | `stretch_goals::claim_stretch_goal_reward` | After `StretchGoalReward.claimed = true` and the payout transfer | Backer claimed a granted reward |
| `VestingTimelineCreatedEvent` | `timeline_id: u64`, `name: String`, `cliff_duration: u64`, `vesting_duration: u64`, `admin: Address`, `now: u64` | `vesting`'s admin-timeline path | After writing the new `VestingTimeline` | Reusable vesting template created |
| `VestingTimelineUpdatedEvent` | `timeline_id: u64`, `cliff_duration: u64`, `vesting_duration: u64`, `admin: Address`, `now: u64` | `vesting`'s admin-timeline path | After writing the updated timeline | Vesting template edited |
| `VestingScheduleReleasedEvent` | `timeline_id: u64`, `tranche_index: u64`, `unlock_amount: i128`, `now: u64` | `vesting`'s admin tranche-release path | After marking the tranche released | An admin-managed vesting tranche released |
| `CreatorVestingCreatedEvent` | `campaign_id: u64`, `creator: Address`, `token: Address`, `total_amount: i128`, `start_time: u64`, `cliff_timestamp: u64`, `end_timestamp: u64`, `initial_unlock_bps: u32`, `initial_unlock_amount: i128`, `campaign_raised: i128`, `timestamp: u64` | `vesting::create_creator_vesting` | After writing the new `CreatorVesting` | Creator committed a campaign's raise to a payout schedule |
| `CreatorVestingReleasedEvent` | `campaign_id: u64`, `creator: Address`, `token: Address`, `amount: i128`, `total_released: i128`, `vested_to_date: i128`, `remaining_amount: i128`, `completed: bool`, `timestamp: u64` | `vesting::release_creator_vesting` | After the release transfer and `CreatorVesting.released_amount` write | Creator drew down vested funds |
| `CreatorVestingRevokedEvent` | `campaign_id: u64`, `creator: Address`, `admin: Address`, `vested_amount: i128`, `unreleased_amount: i128`, `forfeited_amount: i128`, `timestamp: u64` | `vesting::revoke_creator_vesting` | After `CreatorVesting.status = Revoked` | Admin froze a schedule |
| `CampaignFiatGoalSetEvent` | `campaign_id: u64`, `merchant: Address`, `token: Address`, `currency: String`, `goal_amount: i128`, `decimals: u32`, `oracle: Address`, `price: i128`, `price_decimals: u32`, `token_goal_estimate: i128`, `deadline: u64`, `timestamp: u64` | `fiat_goals::set_campaign_fiat_goal` | After writing the new `CampaignFiatGoal` | Fiat-pegged funding target published |
| `FiatContributionEvent` | `campaign_id: u64`, `contributor: Address`, `token: Address`, `token_amount: i128`, `fiat_amount: i128`, `currency: String`, `price: i128`, `price_decimals: u32`, `raised_amount: i128`, `goal_amount: i128`, `progress_bps: u32`, `timestamp: u64` | `fiat_goals::record_fiat_contribution` | After the oracle valuation and `CampaignFiatGoal.raised_amount` write | Contribution valued and credited against a fiat peg |
| `FiatGoalReachedEvent` | `campaign_id: u64`, `merchant: Address`, `currency: String`, `goal_amount: i128`, `raised_amount: i128`, `raised_tokens: i128`, `contribution_count: u32`, `timestamp: u64` | `fiat_goals::record_fiat_contribution` (once, on the contribution that first meets the target) | Same call as `FiatContributionEvent`, immediately after, only the first time the target is met | Fiat-pegged goal reached |
| `FiatGoalQuoteEvent` | `campaign_id: u64`, `token: Address`, `currency: String`, `price: i128`, `price_decimals: u32`, `raised_amount: i128`, `goal_amount: i128`, `remaining_amount: i128`, `tokens_required: i128`, `progress_bps: u32`, `timestamp: u64` | `fiat_goals::refresh_campaign_fiat_quote` | After re-reading the oracle | Merchant refreshed the on-ledger fiat valuation |
| `FiatGoalClosedEvent` | `campaign_id: u64`, `caller: Address`, `currency: String`, `goal_amount: i128`, `raised_amount: i128`, `raised_tokens: i128`, `progress_bps: u32`, `goal_reached: bool`, `timestamp: u64` | `fiat_goals::close_campaign_fiat_goal` | After `CampaignFiatGoal.status = Closed` | Fiat peg wound down |

## Multi-sig withdrawals & search telemetry (shade)

| Event | Data fields | Emitting function | Emission point | State transition |
|---|---|---|---|---|
| `MultisigThresholdSetEvent` | `token: Address`, `threshold: i128`, `admin: Address`, `timestamp: u64` | `multisig_withdrawal::set_multisig_threshold` | After writing `MultiSigKey::MultiSigThreshold` | Multi-sig threshold configured for a token |
| `MultisigConfiguredEvent` | `signers: Vec<Address>`, `quorum: u32`, `admin: Address`, `timestamp: u64` | `multisig_withdrawal::configure_multisig` | After writing the signer list and quorum | Signer set/quorum replaced |
| `WithdrawalProposedEvent` | `proposal_id: u64`, `merchant: Address`, `token: Address`, `amount: i128`, `recipient: Address`, `quorum: u32`, `now: u64` | `multisig_withdrawal::propose_withdrawal` | After writing the new `WithdrawalProposal` | Large withdrawal proposed |
| `WithdrawalApprovedEvent` | `proposal_id: u64`, `signer: Address`, `approvals_so_far: u32`, `quorum_required: u32`, `timestamp: u64` | `multisig_withdrawal::approve_withdrawal` | After recording the signer's approval | A signer approved a pending proposal |
| `WithdrawalExecutedEvent` | `proposal_id: u64`, `merchant: Address`, `token: Address`, `amount: i128`, `recipient: Address`, `signer: Address`, `now: u64` | `multisig_withdrawal::approve_withdrawal` (quorum-reached branch) | After the payout transfer and `WithdrawalProposal.status = Executed`, in the same call as the approval that reached quorum | Quorum reached; funds transferred |
| `WithdrawalCancelledEvent` | `proposal_id: u64`, `caller: Address`, `timestamp: u64` | `multisig_withdrawal::cancel_withdrawal` | After `WithdrawalProposal.status = Cancelled` | Proposal cancelled before execution |
| `InvoiceSearchExecutedEvent` | `caller: Address`, `count: u32`, `has_next: bool`, `timestamp: u64` | `search::search_invoices_paginated` | After building the result page | Telemetry: an invoice search ran |
| `WithdrawalProposalSearchEvent` | `caller: Address`, `count: u32`, `timestamp: u64` | `search::search_withdrawal_proposals` | After building the result `Vec` | Telemetry: a withdrawal-proposal search ran |

## Campaign analytics exports (shade)

| Event | Data fields | Emitting function | Emission point | State transition |
|---|---|---|---|---|
| `CampaignStatsUpdatedEvent` | `campaign_id: u64`, `backer: Address`, `amount: i128`, `is_new_backer: bool`, `pledge_count: u32`, `backer_count: u32`, `tracked_raised: i128`, `average_pledge: i128`, `largest_pledge: i128`, `smallest_pledge: i128`, `timestamp: u64` | `analytics`'s contribution-tracking path | After updating the running `CampaignStats` aggregate | Every tracked contribution updates the live aggregate |
| `AnalyticsExportEvent` | `export_id: u64`, `campaign_id: u64`, `creator: Address`, `merchant_id: u64`, `token: Address`, `format: ExportFormat`, `sequence: u32`, `period_start: u64`, `period_end: u64`, `campaign_raised: i128`, `campaign_deadline: u64`, `campaign_active: bool`, `total_raised: i128`, `pledge_count: u32`, `backer_count: u32`, `average_pledge: i128`, `largest_pledge: i128`, `smallest_pledge: i128`, `first_pledge_at: u64`, `last_pledge_at: u64`, `period_raised: i128`, `period_pledges: u32`, `period_backers: u32`, `timestamp: u64` | `analytics::export_campaign_analytics` (via `publish_analytics_export_event`, which takes the whole `AnalyticsExport` record rather than positional args, to avoid a mistyped call silently transposing fields) | After writing the new `AnalyticsExport` | Creator snapshotted the campaign's analytics |

## `account` contract

All from [`contracts/account/src/events.rs`](../../contracts/account/src/events.rs).

| Event | Topic | Data fields | Notes |
|---|---|---|---|
| `AccountInitializedEvent` | derived: `account_initialized_event` | `merchant: Address`, `merchant_id: u64`, `timestamp: u64` | Emitted from `initialize` |
| `AccountRestrictedEvent` | **explicit:** `account_restricted` | `status: bool`, `timestamp: u64` | The one event in the workspace with an explicit `topics = [...]` override |
| `AccountVerifiedEvent` | derived: `account_verified_event` | `timestamp: u64` (published via `#[contractevent(data_format = "single-value")]`, so the payload is the bare `u64`, not a field map) | Also aliased as `AccountVerified` for use in tests |
| `TokenAddedEvent` | derived | `token: Address`, `timestamp: u64` | Distinct type from `shade`'s own `TokenAddedEvent` — same name, different contract, different struct |
| `RefundProcessedEvent` | derived | `token: Address`, `amount: i128`, `recipient: Address`, `timestamp: u64` | Emitted by the account contract's `refund` entry point, which `shade`'s `invoice::refund_invoice`/`claim_refund` call into via `MerchantAccountRefundClient` |
| `WithdrawalToEvent` | derived | `token: Address`, `merchant: Address`, `recipient: Address`, `amount: i128`, `timestamp: u64` | Emitted by `withdraw_to`, which `shade`'s auto-withdrawal and multi-sig components call into |
| `PendingWithdrawalCreatedEvent` | derived | `id: u64`, `token: Address`, `amount: i128`, `recipient: Address`, `initiator: Address`, `timestamp: u64` | Account-contract-local pending-withdrawal flow |
| `WithdrawalApprovedEvent` | derived | `id: u64`, `approver: Address`, `timestamp: u64` | Distinct from `shade`'s own multi-sig `WithdrawalApprovedEvent` |
| `WithdrawalExecutedEvent` | derived | `id: u64`, `timestamp: u64` | Distinct from `shade`'s own `WithdrawalExecutedEvent` |

## `escrow` contract

[`contracts/escrow/src/lib.rs`](../../contracts/escrow/src/lib.rs) defines a single event.

| Event | Data fields | Notes |
|---|---|---|
| `ReleasedEvent` | `buyer: Address`, `seller: Address`, `amount: i128` | Emitted when the standalone escrow releases funds to the seller. Unrelated to `shade`'s own `EscrowReleasedEvent` — different contract, no shared field set (no `escrow_id`, `token`, `fee`, or `timestamp` here). |

## `ticketing` contract

Selected events from [`contracts/ticketing/src/lib.rs`](../../contracts/ticketing/src/lib.rs) (the standalone ticketing contract, not `shade`'s `event` component):

| Event | Data fields (partial; see source) |
|---|---|
| `EventCreatedEvent` | `event_id`, `organizer`, `name`, … |
| `TicketIssuedEvent` | `ticket_id`, `event_id`, `holder`, … |
| `TicketCheckedInEvent` | `ticket_id`, `event_id`, `holder`, … |
| `TicketTransferedEvent` *(sic — misspelled in source)* | `ticket_id`, `event_id`, `old_holder`, … |
| `TicketResoldEvent` | `ticket_id`, `event_id`, `seller`, … |
| `TicketRefundedEvent` | `ticket_id`, `event_id`, `holder`, … |
| `EventCancelledEvent` | `event_id`, `organizer`, `timestamp` |
| `WaitlistJoinedEvent` | `event_id`, `applicant`, `position`, … |
| `WaitlistAssignedEvent` | `event_id`, `assignee`, `ticket_id`, … |
| `TierCreatedEvent` | `tier_id`, `event_id`, `name`, … |
| `ResaleConfiguredEvent` | `event_id`, `organizer`, `payment_token`, … |

## `ticketing_factory` contract

| Event | Data fields |
|---|---|
| `EventContractDeployedEvent` | `ref_id: u64`, `contract: Address`, `organizer: Address`, `timestamp: u64` |

## `subscription` contract

From [`contracts/subscription/src/types.rs`](../../contracts/subscription/src/types.rs) — note this workspace's standalone `subscription` contract defines its own event types, unrelated to `shade`'s built-in subscription component.

| Event | Data fields |
|---|---|
| `SubRenewed` | `subscription_id: u64`, `plan_id: u64`, `customer: Address`, `timestamp: u64` |
| `SubExpired` | `subscription_id: u64`, `plan_id: u64`, `customer: Address`, `timestamp: u64` |

## `crowdfund` contract

A large, independent event set in [`contracts/crowdfund/src/lib.rs`](../../contracts/crowdfund/src/lib.rs) covering the standalone crowdfund contract's affiliate/referral, guardian-recovery, KYC-gating, and badge systems — none of which exist in `shade`'s own crowdfunding components. Selected highlights:

| Event | Data fields |
|---|---|
| `CampaignExecutedEvent` | `amount: i128` |
| `RefundClaimedEvent` | `contributor: Address`, `amount: i128` |
| `StretchGoalReachedEvent` | `milestone_index: u32`, `threshold: i128` |
| `DiscountAppliedEvent` | `contributor`, `original_amount`, `discounted_amount`, `discount_bps` |
| `MilestoneVoteCastEvent` | `index: u32`, `voter: Address`, `approve: bool`, `weight: i128` |
| `AffiliateRegisteredEvent` | `affiliate: Address`, `code_hash: BytesN<32>`, `commission_bps: u32` |
| `ReferralUsedEvent` | `contributor`, `affiliate`, `code_hash`, `contribution_amount`, `commission_amount` |
| `RecoveryInitiatedEvent` / `RecoveryApprovedEvent` / `RecoveryExecutedEvent` / `RecoveryCancelledEvent` | Guardian-based organizer-recovery flow |
| `KycRequirementSetEvent` / `KycVerificationSetEvent` | Per-campaign KYC gating, independent of `shade`'s `kyc` component |
| `BadgeAwardedEvent` / `BadgeConfigSetEvent` | Backer badge system |

See the source file for the complete list — it is the largest single event set in the workspace and duplicating it in full here would drift out of sync quickly; this table exists to make clear the *domain coverage* (affiliate, recovery, KYC, badges) that has no `shade` equivalent.

## `crowdfund_factory` contract

| Event | Data fields |
|---|---|
| `CampaignDeployedEvent` | `campaign_id: u64`, `contract: Address`, `organizer: Address`, `deployed_at: u64` |
| `ReviewerGrantedEvent` / `ReviewerRevokedEvent` | `reviewer: Address`, `granted_by`/`revoked_by: Address` |
| `CampaignProposalCreatedEvent` | `proposal_id`, `organizer`, `token`, `goal`, `deadline`, `created_at` |
| `CampaignProposalApprovedEvent` / `CampaignProposalRejectedEvent` | Reviewer decision on a proposed campaign |
| `CampaignProposalExecutedEvent` | `proposal_id`, `campaign_id`, `contract`, `executed_at` |
| `DaoMemberAddedEvent` / `DaoMemberRemovedEvent` | `member: Address` |
| `DaoProposalCreatedEvent` / `DaoVoteCastEvent` / `DaoProposalExecutedEvent` / `DaoProposalRejectedEvent` | Factory-level DAO upgrade voting, independent of `shade`'s own `governance` component |

## Topic conventions

Because every event here publishes under a single derived-or-explicit topic Symbol (see [How Soroban events work here](#how-soroban-events-work-here-read-before-the-tables)), "filtering by topic" in the way a multi-topic event system would support does not apply. What you actually have available to filter on:

1. **The emitting contract's address** — every event carries the address of the contract that emitted it (returned alongside the event by `env.events().all()` in tests, and by the RPC/Horizon event-streaming APIs in production).
2. **The event's topic Symbol** (the struct name in snake_case, or the explicit override) — lets you distinguish "this was an `InvoicePaidEvent`" from "this was a `PaymentSplitRoutedEvent`" without inspecting the payload.
3. **Fields inside the data payload** — `merchant_id`, `invoice_id`, `campaign_id`, `token`, and so on are ordinary named fields in the event's data map, not part of the topic. Matching "all payments for one merchant" means decoding each `InvoicePaidEvent`'s data and comparing its `merchant_id` field — there is no server-side topic-based filter for this in the contract itself.

### Filter examples

**All payments for one merchant** — filter the stream to events with topic `invoice_paid_event` emitted by the `shade` contract address, then decode each one's data and keep those whose `merchant_id` field matches:

```rust
// Illustrative decode logic, following the same pattern the test helper
// `assert_latest_paid_event` in contracts/shade/src/tests/test_payment.rs uses.
use soroban_sdk::{Map, Symbol, TryIntoVal, Val};

let target_merchant_id: u64 = 1;
for (contract_id, _topics, data) in env.events().all().iter() {
    if contract_id != shade_contract_id {
        continue;
    }
    let Ok(fields): Result<Map<Symbol, Val>, _> = data.try_into_val(&env) else { continue };
    let Some(merchant_id_val) = fields.get(Symbol::new(&env, "merchant_id")) else { continue };
    let merchant_id: u64 = merchant_id_val.try_into_val(&env).unwrap();
    if merchant_id == target_merchant_id {
        // this is an InvoicePaidEvent (or any other event carrying merchant_id) for our merchant
    }
}
```

**All status changes for one invoice** — the events that mark an invoice status transition are `InvoiceCreatedEvent`, `InvoicePaidEvent`, `InvoiceCancelledEvent`, `InvoiceRefundedEvent`/`InvoicePartiallyRefundedEvent`, and `InvoiceAmendedEvent`; every one of them carries `invoice_id`. Decode each event on the `shade` contract address whose data map contains an `invoice_id` field equal to the target ID, in emission order (see [Ordering guarantees](#precise-semantics)) to reconstruct the invoice's full status history.

## Precise semantics

- **Only successful transactions leave events.** A transaction that panics (any `panic_with_error!` — status/amount/auth/expiry failures, `InsufficientBalance`, etc.) rolls back its entire effect, including every event it would have emitted. A reverted `pay_invoice` call, for example, produces **no** `InvoicePaidEvent`, `PaymentSplitRoutedEvent`, or `PlatformFeeRoutedEvent` — there is nothing to observe from a failed call except the transaction's own failure status.
- **Multiple events in one successful call preserve emission order.** [`env.events().all()`](../../contracts/shade/src/tests/test_payment.rs) returns events in the order they were published, and a single `pay_invoice` call is a concrete example: `PlatformFeeRoutedEvent` (emitted inside `finalize_route`, called from `route_from_payer`) is published before `InvoicePaidEvent` and `PaymentSplitRoutedEvent` (both emitted directly in `pay_invoice_partial_inner`, after the call to `route_from_payer` returns) — so the exact order for one full invoice payment is `PlatformFeeRoutedEvent`, then `InvoicePaidEvent`, then `PaymentSplitRoutedEvent`. [`test_payment.rs`](../../contracts/shade/src/tests/test_payment.rs)'s own `assert_latest_paid_event` helper explicitly does **not** assume the `InvoicePaidEvent` is the last event in the list — it searches from the end for the event whose data map has both `merchant_id` and `payer` fields — precisely because more than one event is emitted per call and their relative order, while deterministic, is not "the interesting one is always last."
- **Do not assume stronger guarantees than this.** There is no on-chain event log a contract can read back — events are a write-only, off-chain-observable side channel. A component cannot decide behavior based on "was this event already emitted," and nothing in this workspace does. Order is guaranteed only *within* one successful invocation, not across separate transactions unless the ledger's own transaction ordering is what you're relying on.

## Decoding a real event

Using this repository's own test conventions (`soroban-sdk = "23.4.0"`, per the root `Cargo.toml`) — the exact pattern `contracts/shade/src/tests/test_payment.rs` and `test_invoice.rs` use to assert on emitted events in an integration test:

```rust
use soroban_sdk::testutils::Events; // brings `.events()` into scope on Env
use soroban_sdk::{Map, Symbol, TryIntoVal, Val};

// After calling shade_client.pay_invoice(&customer, &invoice_id):
let events = env.events().all();
assert!(!events.is_empty());

// Each entry is (emitting_contract_address, topics, data).
let (event_contract_id, _topics, data) = events.get(events.len() - 1).unwrap();

// The default encoding is a Map<Symbol, Val> keyed by the struct's field names.
let fields: Map<Symbol, Val> = data.try_into_val(&env).unwrap();

let invoice_id: u64 = fields
    .get(Symbol::new(&env, "invoice_id"))
    .unwrap()
    .try_into_val(&env)
    .unwrap();
let amount: i128 = fields
    .get(Symbol::new(&env, "amount"))
    .unwrap()
    .try_into_val(&env)
    .unwrap();
```

This is exactly the shape `assert_latest_paid_event` and `assert_latest_invoice_event` use in the test suite — nothing here is an invented SDK API; every call (`env.events().all()`, `Map<Symbol, Val>`, `TryIntoVal`, `Symbol::new`) is copied from those test files. Outside a test harness (e.g. reading events from an RPC provider in a client application), the payload arrives as XDR that decodes to the same field-map shape; consult the Stellar SDK for your client language for its own equivalent of `try_into_val`.

## Related pages

- [Architecture overview](overview.md) — the `pay_invoice` lifecycle these events instrument.
- [The component pattern](component-pattern.md) — where in a component's code an event is emitted, and why.
- [Accepting payments](../guides/accepting-payments.md) — consuming these events from a payer-facing client.
- [`ShadeTrait` function reference](shade-interface.md)

← [Back to Reference](README.md)
