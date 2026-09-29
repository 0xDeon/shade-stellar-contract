# Errors reference

Every [`#[contracterror]`](https://docs.rs/soroban-sdk) enum in the workspace, the numeric code each variant raises, the functions that can raise it, the condition that triggers it, and the corrective action. Read this page after a failed transaction to go from an error code to a fix.

## Code uniqueness

Error codes are scoped to the contract that raised them, not global. `1` in [`account`](#account-contracts-src-errorsrs) means `AlreadyInitialized`; `1` in [`shade`](#shade-contracts-shade-src-errorsrs) means `NotAuthorized`. Always pair a numeric code with the contract it came from — the contract ID in the failed transaction tells you which table below to read.

The `shade` crate is the one exception with internal structure: it partitions nine error enums into non-overlapping numeric ranges within the same contract, specifically so a raw code observed on-chain against the `shade` contract ID maps back to exactly one variant. See [`contracts/shade/src/errors.rs#L1-L23`](../../contracts/shade/src/errors.rs#L1-L23):

| Range | Enum |
|---|---|
| 1–99 | `ContractError` |
| 100–119 | `GovernanceError` |
| 120–139 | `EventError` |
| 140–159 | `MultiSigError` |
| 160–179 | `EscrowError` |
| 200–239 | `CampaignError` |
| 240–259 | `StretchGoalError` |
| 260–279 | `VestingError` |
| 280–299 | `FiatGoalError` |
| 300–319 | `AnalyticsError` |

Every other contract in the workspace (`account`, `escrow`, `escrow_factory`, `ticketing`, `ticketing_factory`, `subscription`, `crowdfund`, `crowdfund_factory`) defines exactly one error enum starting at `1`, independent of every other contract's numbering.

## `shade` (`contracts/shade/src/errors.rs`)

The `shade` hub contract is the primary payment-processing entry point most integrators call against. Its errors are split into nine enums because Soroban caps a single `#[contracterror]` enum at 50 variants.

### `ContractError` (codes 1–99)

| Error | Code | Raised by | Trigger | Corrective action |
|---|---:|---|---|---|
| `NotAuthorized` | 1 | Most admin-only and role-gated functions (e.g. `set_fee`, `add_accepted_token`, `pause`, `grant_role`, `restrict_merchant_account`) | Caller does not hold the required role, or `require_auth` fails for the claimed address. | Call with the correct signer — usually the current admin (`get_admin`) or a granted [role](../glossary.md#role). |
| `AlreadyInitialized` | 2 | `initialize` | `initialize` was already called once on this contract instance. | Do not re-initialize; use `get_admin` to confirm the current admin. |
| `NotInitialized` | 3 | Internal reads of `DataKey::Admin` before `initialize` has run | The contract has never been initialized. | Call `initialize` first. |
| `Reentrancy` | 4 | Fund-moving functions guarded by [`reentrancy::enter`/`exit`](../../contracts/shade/src/components/reentrancy.rs) | A guarded function is re-entered before its first call completes (e.g. via a malicious token callback). | Indicates an attempted reentrant call; not user-recoverable. Review the calling transaction for unexpected cross-contract calls. |
| `MerchantAlreadyRegistered` | 5 | `register_merchant` | The calling address already has a `DataKey::MerchantId` entry. | Use `find_merchant_id` or `is_merchant` to check registration before calling `register_merchant` again. |
| `MerchantNotFound` | 6 | `get_merchant`, `set_merchant_status`, `verify_merchant`, `set_merchant_key`, `set_merchant_account`, `set_merchant_accepted_tokens`, `set_merchant_webhook`, and most merchant/invoice/escrow/campaign functions that resolve a merchant by ID or address | The merchant ID exceeds the registered count, or the address has no `DataKey::MerchantId` entry. | Confirm registration with `is_merchant` / `find_merchant_id`, or call `register_merchant` first. |
| `InvalidAmount` | 7 | Most amount-taking functions across invoices, campaigns, escrow, subscriptions, tickets, NFTs, pledges (e.g. `create_invoice`, `pay_invoice_partial`, `create_escrow`) | The supplied amount is zero, negative, or otherwise fails the function's amount validation. | Supply a positive amount consistent with the function's documented constraints. |
| `InvoiceNotFound` | 8 | `get_invoice` and every function that loads an invoice by ID | The invoice ID does not exist. | Confirm the ID from `create_invoice`'s return value or `get_invoices`/`search_invoices_paginated`. |
| `ContractPaused` | 9 | Every state-mutating function guarded by [`pausable::assert_not_paused`](../../contracts/shade/src/components/pausable.rs) (does **not** guard `upgrade` — see [Upgradeability](../architecture/upgradeability.md)) | Admin has called `pause` and not yet called `unpause`. | Wait for `unpause`, or contact the admin. Check with `is_paused`. |
| `ContractNotPaused` | 10 | `unpause` | `unpause` called while the contract is not paused. | No action needed; the contract is already active. |
| `MerchantKeyNotFound` | 11 | `get_merchant_key`, `create_invoice_signed` (via `verify_invoice_signature`) | The merchant never called `set_merchant_key`. | Call `set_merchant_key` before using signed-invoice creation. See [Signed invoices](../security/signatures.md). |
| `TokenNotAccepted` | 12 | `set_merchant_accepted_tokens`, `create_invoice` (token whitelist check) | The token is not in the global `DataKey::AcceptedTokens` list. | Ask the admin to call `add_accepted_token` for this token first. |
| `NonceAlreadyUsed` | 14 | `create_invoice_signed` | The 32-byte nonce was already consumed by a prior signed invoice from this merchant. | Use a fresh nonce; see [Nonce and replay protection](../security/signatures.md#nonce-and-replay-protection). |
| `InvalidInvoiceStatus` | 16 | `finalize_invoice`, `pay_invoice`, `refund_invoice`, `claim_refund`, `void_invoice` | The invoice is not in the status the operation requires (e.g. refunding an invoice that was never paid). | Read `get_invoice` and confirm `status` matches what the operation expects before calling. |
| `RefundPeriodExpired` | 17 | `refund_invoice` | The refund window (measured from `date_paid`) has elapsed. | No further refund is possible through this path once the window closes. |
| `WasmHashNotSet` | 18 | `upgrade`, merchant-account deployment inside `register_merchant` | `set_account_wasm_hash` (for deployment) or a valid upgrade target has not been configured. | Admin must call `set_account_wasm_hash` before merchants can register, or supply a valid hash to `upgrade`. |
| `MerchantAccountNotSet` | 20 | `get_merchant_account` | No `DataKey::MerchantAccount` entry exists for this merchant ID. | Call `set_merchant_account` first. |
| `InvalidInterval` | 21 | `create_subscription_plan` | The billing interval is zero or otherwise invalid. | Supply a positive interval in seconds. |
| `PlanNotFound` | 22 | `get_subscription_plan`, `subscribe`, `deactivate_plan` | The plan ID does not exist. | Confirm the ID from `create_subscription_plan`'s return value. |
| `PlanNotActive` | 23 | `subscribe` | The plan was deactivated via `deactivate_plan`. | Cannot subscribe to a deactivated plan; the merchant must create a new plan. |
| `SubscriptionNotFound` | 24 | `get_subscription`, `charge_subscription`, `cancel_subscription` | The subscription ID does not exist. | Confirm the ID from `subscribe`'s return value. |
| `SubscriptionNotActive` | 25 | `charge_subscription` | The subscription was already cancelled. | No further charges are possible on a cancelled subscription. |
| `ChargeTooEarly` | 26 | `charge_subscription` | The plan's billing interval has not yet elapsed since the last charge. | Wait until the interval elapses; check `Subscription`'s last-charge timestamp via `get_subscription`. |
| `InvoiceExpired` | 27 | `pay_invoice`, `pay_invoice_partial` | The current ledger timestamp exceeds the invoice's `expires_at`. | The merchant must create a new invoice; expired invoices cannot be paid. |
| `InvoiceNotPaid` | 28 | Internal invoice-settlement paths | An operation that requires a paid invoice was attempted on one that is not paid. | Confirm invoice status is `Paid` via `get_invoice` before calling. |
| `PayerNotAvailable` | 29 | Internal invoice-settlement paths | No payer is recorded on the invoice at a point where one is required. | Indicates the invoice was never paid; not directly user-triggerable through a normal flow. |
| `InsufficientBalance` | 30 | `pay_invoice`, `pay_invoice_partial`, `charge_subscription`, `subscribe` | The payer/customer's token balance or SEP-41 allowance is insufficient. | Fund the paying address, or increase the token allowance for the contract. |
| `MerchantNotActive` | 32 | `create_invoice`, `set_merchant_accepted_tokens`, `remove_merchant_accepted_token` | `set_merchant_status` has set `active = false` for this merchant. | Ask the admin to call `set_merchant_status(admin, merchant_id, true)`. |
| `InvalidDescription` | 33 | `create_invoice`, `create_fiat_invoice`, `create_invoice_draft` | The description string fails length or content validation. | Shorten or otherwise correct the description. |
| `OracleNotConfigured` | 34 | `get_token_oracle`, `create_fiat_invoice`, `resolve_invoice_amount` | No `OracleConfig` has been set for the token via `set_token_oracle`. | Admin must call `set_token_oracle` for this token before fiat pricing can resolve. |
| `OraclePriceUnavailable` | 35 | `resolve_invoice_amount` and other fiat-resolution paths | The configured oracle contract returned no usable price. | Retry once the oracle publishes a price, or contact the oracle operator. |
| `TokenNotAcceptedByMerchant` | 41 | `pay_invoice`, `create_subscription_plan` | The token is globally accepted but not in this merchant's own accepted-tokens list (see [`is_token_accepted_for_merchant`](../../contracts/shade/src/components/merchant.rs#L444-L460)). | Merchant must call `set_merchant_accepted_tokens` to include this token, or pay in a token the merchant already accepts. |
| `FeeUpdateTooEarly` | 42 | `execute_fee` | The fee time-lock configured by `propose_fee` has not elapsed. | Wait for the time-lock to elapse before calling `execute_fee`. |
| `NoPendingFeeUpdate` | 43 | `execute_fee`, `get_pending_fee` | No fee change is currently staged for this token. | Call `propose_fee` first. |
| `InvalidSwapPath` | 44 | `validate_payment_payload` | The `PaymentPayload`'s swap route is malformed. | Correct the `PaymentRoute` before resubmitting. |
| `InvalidSlippage` | 45 | `validate_payment_payload` | The requested slippage basis points exceed the accepted maximum. | Lower the slippage tolerance. |
| `EscrowNotFound` | 46 | `get_escrow`, `fund_escrow`, `release_escrow`, `refund_escrow` | The escrow ID does not exist. | Confirm the ID from `create_escrow`'s return value. |
| `InvalidEscrowStatus` | 47 | `fund_escrow`, `release_escrow`, `refund_escrow` | The escrow is not in the status the operation requires. | Read `get_escrow` and confirm `status` before calling. |
| `NotFound` | 48 | Generic fallback for lookups without a dedicated code | A record referenced by a less common lookup path does not exist. | Confirm the ID/key exists via the corresponding getter before the write call. |
| `BridgeDepositProcessed` | 49 | `record_bridge_deposit` | This `source_tx_id` was already credited (idempotency guard). | No action — the deposit has already been recorded; check `get_bridge_deposit`/`is_bridge_deposit_processed`. |

### `GovernanceError` (codes 100–119)

| Error | Code | Raised by | Trigger | Corrective action |
|---|---:|---|---|---|
| `NotGovMember` | 100 | `propose_upgrade`, `vote_on_upgrade` | Caller is not registered via `add_gov_member`. | Admin must call `add_gov_member` for this address first. |
| `GovNotConfigured` | 101 | `propose_upgrade` | `set_governance_config` has not been called. | Admin must call `set_governance_config` before proposals can be opened. |
| `InvalidGovConfig` | 102 | `set_governance_config` | `voting_period` is zero, or `quorum_bps` exceeds 10,000 (100%). | Supply a positive voting period and a quorum between 0 and 10,000 bps. |
| `ProposalNotFound` | 103 | `get_upgrade_proposal`, `vote_on_upgrade`, `finalize_upgrade` | The proposal ID does not exist. | Confirm the ID from `propose_upgrade`'s return value. |
| `ProposalNotActive` | 104 | `vote_on_upgrade`, `finalize_upgrade` | The proposal has already been finalized (executed or defeated). | No further action possible on a finalized proposal. |
| `VotingClosed` | 105 | `vote_on_upgrade` | The voting window (`voting_ends_at`) has passed. | Voting has ended; wait for `finalize_upgrade` to be called. |
| `VotingStillOpen` | 106 | `finalize_upgrade` | The voting window has not yet closed. | Wait until `voting_ends_at` before finalizing. |
| `AlreadyVoted` | 107 | `vote_on_upgrade` | This council member already cast a vote on this proposal. | Each member may vote once per proposal; no correction possible. |

### `EventError` (codes 120–139)

These are the ticketing-domain errors surfaced through the `shade` hub's ticketing entry points (`create_event`, `purchase_ticket`, `resell_ticket`, etc.).

| Error | Code | Raised by | Trigger | Corrective action |
|---|---:|---|---|---|
| `EventNotFound` | 120 | `get_event`, `purchase_ticket`, `configure_dynamic_pricing`, `cancel_event_and_batch_refund` | The event ID does not exist. | Confirm the ID from `create_event`'s return value. |
| `EventSoldOut` | 121 | `purchase_ticket`, `purchase_tickets_bulk` | Ticket capacity has been reached. | No further tickets can be sold for this event. |
| `InvalidCapacity` | 122 | `create_event` | Capacity is zero or otherwise invalid. | Supply a positive capacity. |
| `InvalidEventDate` | 123 | `create_event` | The event date fails validation (e.g. in the past). | Supply a future `event_date`. |
| `InvalidRoyaltyBps` | 124 | `create_event` | `royalty_bps` exceeds the accepted maximum basis points. | Supply a royalty in the valid basis-point range. |
| `TicketNotFound` | 125 | `get_ticket`, `resell_ticket` | The ticket ID does not exist. | Confirm the ID from `purchase_ticket`'s return value. |
| `NotTicketOwner` | 126 | `resell_ticket` | The caller does not own the ticket being resold. | Only the current ticket holder may resell it. |
| `InvalidResalePrice` | 127 | `resell_ticket` | Resale price falls outside the enforced 0.5x–2x bound of the original price. | Set a resale price within the enforced band. |
| `TicketNotListed` | 128 | Secondary-market resale paths | An operation expected an active resale listing that does not exist. | List the ticket for resale first. |
| `TicketAlreadyListed` | 129 | Secondary-market resale paths | The ticket already has an active resale listing. | Cancel the existing listing before creating a new one. |

### `MultiSigError` (codes 140–159)

| Error | Code | Raised by | Trigger | Corrective action |
|---|---:|---|---|---|
| `BelowMultiSigThreshold` | 140 | `propose_withdrawal` | The requested amount is below the configured per-token threshold. | Withdraw directly if multi-sig is not required, or raise the amount above the threshold. |
| `MultiSigSignersNotSet` | 141 | `propose_withdrawal`, `approve_withdrawal` | `configure_multisig` has never been called. | Admin must call `configure_multisig` first. |
| `InvalidQuorum` | 142 | `configure_multisig` | Quorum is zero or exceeds the number of signers. | Supply a quorum between 1 and the signer count. |
| `NotASigner` | 143 | `approve_withdrawal` | Caller is not in the registered signer list. | Only registered signers (see `configure_multisig`) may approve. |
| `AlreadyApproved` | 144 | `approve_withdrawal` | This signer already approved the given proposal. | Each signer may approve once per proposal. |
| `ProposalNotFound` | 145 | `get_withdrawal_proposal`, `approve_withdrawal`, `cancel_withdrawal` | The proposal ID does not exist. | Confirm the ID from `propose_withdrawal`'s return value. |
| `ProposalNotPending` | 146 | `approve_withdrawal`, `cancel_withdrawal` | The proposal already executed or was cancelled. | No further action possible on a non-pending proposal. |
| `QuorumNotReached` | 147 | Internal execution check after `approve_withdrawal` | Approvals below quorum when execution was attempted. | Wait for additional signer approvals. |
| `NotProposer` | 148 | `cancel_withdrawal` | Caller is neither the original proposer nor admin. | Only the proposing merchant or admin may cancel. |
| `ThresholdNotSet` | 149 | `get_multisig_threshold`, `propose_withdrawal` | `set_multisig_threshold` has not been called for this token. | Admin must call `set_multisig_threshold` first. |

### `EscrowError` (codes 160–179)

Distinct from the standalone [`escrow` contract](#escrow-contractsescrowsrcerrorsrs) — these cover the `shade` hub's own expired-refund path for escrow-linked invoices.

| Error | Code | Raised by | Trigger | Corrective action |
|---|---:|---|---|---|
| `EscrowNotExpired` | 160 | Expired-refund claim path | The escrow invoice has not yet reached its expiration timestamp. | Wait until expiration before claiming a refund. |
| `EscrowAlreadyRefunded` | 161 | Expired-refund claim path | The escrow invoice has already been fully refunded. | No further refund is possible. |

### `CampaignError` (codes 200–239)

| Error | Code | Raised by | Trigger | Corrective action |
|---|---:|---|---|---|
| `CampaignNotFound` | 200 | `get_campaign`, `update_campaign`, `set_campaign_active`, `add_campaign_tag`, `record_campaign_contribution` | The campaign ID does not exist. | Confirm the ID from `create_campaign`'s return value. |
| `CampaignCategoryNotFound` | 201 | `create_campaign`, `update_campaign_category`, `get_campaign_category` | The category ID does not exist. | Confirm the ID from `create_campaign_category`'s return value. |
| `CampaignCategoryAlreadyExists` | 202 | `create_campaign_category` | A category with this name was already registered. | Use `get_campaign_categories` to find the existing category instead. |
| `CampaignCategoryInactive` | 203 | `create_campaign` | The referenced category was deactivated. | Choose an active category. |
| `CampaignTagNotFound` | 204 | `create_campaign`, `add_campaign_tag`, `remove_campaign_tag` | The tag ID does not exist. | Confirm the ID from `create_campaign_tag`'s return value. |
| `CampaignTagAlreadyExists` | 205 | `create_campaign_tag` | A tag with this name was already registered. | Use `get_campaign_tags` to find the existing tag instead. |
| `InvalidCampaignGoal` | 206 | `create_campaign` | `goal_amount` is not positive. | Supply a positive goal amount. |
| `InvalidCampaignDeadline` | 207 | `create_campaign` | `deadline` is not in the future. | Supply a future deadline timestamp. |
| `CampaignInactive` | 208 | Contribution and management paths | The campaign was deactivated via `set_campaign_active`. | Reactivate the campaign, or choose an active one. |
| `NotCampaignMerchant` | 209 | `update_campaign`, `set_campaign_active`, `add_campaign_tag`, `remove_campaign_tag`, `deactivate_plan` | Caller is not the merchant that owns the campaign/plan. | Only the owning merchant may perform this operation. |
| `CampaignExpired` | 210 | `record_campaign_contribution` | The campaign's deadline has passed. | No further contributions are accepted after the deadline. |
| `CampaignEnded` | 211 | Campaign close-out paths | The campaign was already closed out. | No further action possible on a closed campaign. |
| `CampaignNotActive` | 212 | Contribution and management paths | The campaign is not currently active. | Check `get_campaign` status before calling. |
| `AffiliateNotFound` | 213 | Affiliate commission paths | The referenced affiliate is not registered on this campaign. | Register the affiliate before crediting commission. |
| `InvalidRewardTier` | 214 | Reward-tier selection paths | The tier index does not exist on this campaign. | Confirm the tier index against the campaign's configured tiers. |
| `PledgeBelowTierMinimum` | 215 | Reward-tier selection paths | Pledge amount is below the selected tier's minimum. | Increase the pledge or select a lower tier. |
| `RewardTierAtCapacity` | 216 | Reward-tier selection paths | The selected tier has reached its backer limit. | Select a different tier. |
| `BackerRewardAlreadyFulfilled` | 217 | Reward fulfillment paths | This backer's reward was already marked fulfilled. | No further action needed. |
| `BackerRewardNotFulfilled` | 218 | Reward-dependent paths | An operation requires a fulfilled reward that has not been fulfilled. | Fulfill the reward first. |
| `NotBacker` | 219 | Backer-only paths (milestone voting, perk claims) | The caller has not backed this campaign. | Only recorded backers may perform this operation. |
| `PerkNotFound` | 220 | Perk claim paths | The referenced perk does not exist on the tier. | Confirm the perk index against the tier's configured perks. |
| `PerkAlreadyClaimed` | 221 | Perk claim paths | This perk was already claimed by the backer. | No further action needed. |
| `InvalidTierOrdering` | 222 | Reward-tier configuration | Reward tiers were not supplied in ascending minimum-pledge order. | Sort tiers by ascending `min_pledge` before submitting. |
| `NftError` | 223 | NFT-reward integration paths | A generic NFT reward operation failed. | Check the underlying NFT collection state via `get_nft_collection`. |
| `CommentNotFound` | 224 | Comment moderation paths | The referenced comment does not exist. | Confirm the comment ID. |
| `EmptyComment` | 225 | Comment creation paths | The comment body is empty. | Supply non-empty comment content. |
| `InvalidCommentStatus` | 226 | Comment moderation paths | The comment is not in a state that permits the operation. | Check the comment's current status first. |
| `VotingNotFound` | 227 | Hard-cap voting paths | The referenced hard-cap vote does not exist. | Confirm the vote was opened first. |
| `VotingNotActive` | 228 | Hard-cap voting paths | The hard-cap vote is not currently open. | Wait for a vote to open, or check its status. |
| `AlreadyVoted` | 229 | Hard-cap voting paths | This address already cast a hard-cap vote. | Each address may vote once. |

> **Note:** `CampaignError::AlreadyVoted` (229) and `GovernanceError::AlreadyVoted` (107) share a variant name but are different codes in different enums — always read the code alongside its enum, never the name alone.

### `StretchGoalError` (codes 240–259)

| Error | Code | Raised by | Trigger | Corrective action |
|---|---:|---|---|---|
| `StretchGoalNotFound` | 240 | Stretch-goal lookup/claim paths | No stretch goal exists for the supplied ID. | Confirm the goal ID against the campaign's configured goals. |
| `StretchGoalNotUnlocked` | 241 | Reward-claim paths | The goal has not been unlocked yet. | Wait until the campaign's raise reaches the goal's target. |
| `GoalAlreadyUnlocked` | 242 | Goal-management paths | The goal already left the `Pending` state. | No further action possible; the goal is already unlocked. |
| `RewardAlreadyClaimed` | 243 | Reward-claim paths | This backer already claimed the reward for this goal. | No further action needed. |
| `RewardNotFound` | 244 | Reward-claim paths | No reward has been granted to this backer for this goal. | The backer must be granted a reward before claiming. |
| `RewardAlreadyGranted` | 245 | Reward-grant paths | A reward was already granted to this backer for this goal. | No further action needed. |
| `TargetBelowBaseGoal` | 246 | Goal creation | The goal's target does not exceed the campaign's base funding goal. | Set a target above the campaign's base goal. |
| `TargetNotIncreasing` | 247 | Goal creation | Stretch goal targets must be strictly increasing per campaign. | Set a target higher than the previous goal's target. |
| `TargetNotReached` | 248 | Goal unlock paths | The campaign has not raised enough to unlock this goal. | Wait until the campaign's raise reaches the target. |
| `GoalCancelled` | 249 | Goal-management and claim paths | The goal was cancelled and can no longer be unlocked or claimed. | No further action possible on a cancelled goal. |
| `NotGoalOwner` | 250 | Goal-management paths | Caller is not the campaign's owning merchant. | Only the owning merchant may manage this goal. |
| `TooManyStretchGoals` | 251 | Goal creation | The campaign already has the maximum number of stretch goals. | Remove or consolidate existing goals before adding another. |

### `VestingError` (codes 260–279)

| Error | Code | Raised by | Trigger | Corrective action |
|---|---:|---|---|---|
| `VestingNotFound` | 260 | Vesting lookup/release paths | No vesting schedule exists for the supplied campaign. | Confirm the schedule was created for this campaign. |
| `VestingAlreadyExists` | 261 | Vesting schedule creation | The campaign already has a vesting schedule (schedules are immutable). | Schedules cannot be replaced once created. |
| `NotVestingBeneficiary` | 262 | Release paths | Caller is not the creator this schedule pays out to. | Only the designated beneficiary may release funds. |
| `InvalidVestingDuration` | 263 | Vesting schedule creation | Duration is zero, or the cliff outlasts the whole schedule. | Supply a duration greater than the cliff. |
| `InvalidUnlockBps` | 264 | Vesting schedule creation | `initial_unlock_bps` exceeds 10,000 (100%). | Supply an unlock percentage between 0 and 10,000 bps. |
| `VestingAmountExceedsRaised` | 265 | Vesting schedule creation | The schedule commits more than the campaign has actually raised. | Reduce the vesting amount to at or below the campaign's raise. |
| `NothingToRelease` | 266 | Release paths | Nothing has vested since the last release. | Wait until additional tokens vest per the schedule. |
| `VestingRevoked` | 267 | Release paths | The schedule was revoked; nothing further will vest. | No further releases possible. |
| `VestingCompleted` | 268 | Release paths | Every vested token has already been released. | No further releases possible. |
| `InvalidVestingStart` | 269 | Vesting schedule creation | `start_time` is in the past. | Supply a future or current start time. |
| `TooManyVestingSchedules` | 270 | Vesting schedule creation | This creator already holds the maximum number of vesting schedules. | Complete or release an existing schedule before adding another. |

### `FiatGoalError` (codes 280–299)

| Error | Code | Raised by | Trigger | Corrective action |
|---|---:|---|---|---|
| `FiatGoalNotFound` | 280 | Fiat-goal lookup paths | No fiat-pegged goal exists for the supplied campaign. | Confirm a fiat goal was created for this campaign. |
| `FiatGoalAlreadyExists` | 281 | Fiat-goal creation | The campaign already has a fiat goal (immutable once published). | Goals cannot be replaced once created. |
| `InvalidFiatGoalAmount` | 282 | Fiat-goal creation | The fiat target is not positive. | Supply a positive fiat target amount. |
| `InvalidFiatCurrency` | 283 | Fiat-goal creation | The quote currency is empty or exceeds the accepted length. | Supply a valid, correctly-sized currency code. |
| `InvalidFiatDecimals` | 284 | Fiat-goal creation | `decimals` exceeds the accepted maximum, or oracle decimals would overflow the conversion. | Supply decimals within the accepted range. |
| `FiatGoalClosed` | 285 | Contribution paths | The goal has been wound down; no further contributions are valued. | No further fiat-valued contributions are accepted. |
| `NotFiatGoalOwner` | 286 | Fiat-goal management paths | Caller is not the merchant that owns the pegged campaign. | Only the owning merchant may manage this goal. |
| `FiatValueTooSmall` | 287 | Contribution paths | The contribution is worth less than one minor unit of the goal's currency at the current price. | Increase the contribution amount. |

### `AnalyticsError` (codes 300–319)

| Error | Code | Raised by | Trigger | Corrective action |
|---|---:|---|---|---|
| `AnalyticsExportNotFound` | 300 | Export lookup paths | No export exists under the supplied ID. | Confirm the export ID from its creation call. |
| `NotAnalyticsExportOwner` | 301 | Export management paths | Caller is not the merchant that owns the campaign being exported. | Only the owning merchant may access this export. |
| `NothingToExport` | 302 | Export creation | The campaign has neither tracked contributions nor a raise of its own. | No export is possible for a campaign with no recorded activity. |
| `TooManyExports` | 303 | Export creation | This campaign has already run the maximum number of exports. | Wait, or remove tracking of older exports before creating another. |
| `NoExportsYet` | 304 | Export lookup paths | The campaign has never been exported. | Create an export first. |

## `account` (`contracts/account/src/errors.rs`)

Errors from the per-merchant `account` contract (deployed via the hub's `set_account_wasm_hash`/`register_merchant` flow). This contract's `ContractError` enum is independent of `shade`'s — code `1` here is `AlreadyInitialized`, not `NotAuthorized`.

| Error | Code | Trigger | Corrective action |
|---|---:|---|---|
| `AlreadyInitialized` | 1 | The account contract's `initialize` was already called. | No action needed; the account is already set up. |
| `NotInitialized` | 2 | An operation was attempted before `initialize` ran. | The hub deploys and initializes accounts automatically during `register_merchant`; this indicates a deployment issue, not a merchant-side fix. |
| `NotAuthorized` | 3 | Caller is not the account's owning merchant or manager. | Call from the merchant address the account was deployed for. |
| `InsufficientBalance` | 4 | A withdrawal or disbursement exceeds the account's held balance. | Reduce the requested amount, or wait for further settlement. |
| `AccountRestricted` | 5 | The account was restricted via the hub's `restrict_merchant_account`. | Contact the platform admin; only an admin/manager role can lift the restriction. |
| `InvoiceNotFound` | 6 | The account was asked to act on an invoice ID it has no record of. | Confirm the invoice ID against the hub's `get_invoice`. |
| `InvalidInvoiceStatus` | 7 | The invoice referenced is not in a status the account operation permits. | Check invoice status via the hub's `get_invoice` before calling. |

## `escrow` (`contracts/escrow/src/errors.rs`)

The standalone `escrow` contract (distinct from the `shade` hub's own `EscrowError` range 160–179, and from `ContractError::EscrowNotFound`/`InvalidEscrowStatus` at codes 46–47, which cover the hub's `create_escrow`/`fund_escrow`/`release_escrow`/`refund_escrow` entry points).

| Error | Code | Trigger | Corrective action |
|---|---:|---|---|
| `AlreadyInitialized` | 1 | `initialize` was already called on this escrow instance. | No action needed. |
| `NotInitialized` | 2 | An operation was attempted before `initialize` ran. | Deploy and initialize the escrow contract before use. |
| `InvalidAmount` | 3 | A funding or release amount is zero or negative. | Supply a positive amount. |
| `Overflow` | 4 | An arithmetic operation on escrow balances overflowed. | Reduce the amount; this indicates values outside `i128` safe range. |
| `Overfunded` | 5 | Funding would exceed the escrow's agreed amount. | Fund only up to the remaining amount owed. |
| `InvalidStatus` | 6 | The operation is not valid for the escrow's current status. | Check the escrow's status before calling. |

## `escrow_factory` (`contracts/escrow_factory/src/errors.rs`)

| Error | Code | Trigger | Corrective action |
|---|---:|---|---|
| `AlreadyInitialized` | 1 | `initialize` was already called on the factory. | No action needed. |
| `NotInitialized` | 2 | A deployment was attempted before the factory's `initialize` ran. | Initialize the factory before deploying escrow instances. |

> **Note:** Per [Upgradeability](../architecture/upgradeability.md#hub-upgrades-versus-factory-deployed-wasm), `escrow_factory` has no setter for its deployed WASM hash — it is fixed at `initialize` and cannot be changed afterward.

## `ticketing` (`contracts/ticketing/src/errors.rs`)

`TicketingError` is a standalone enum for the dedicated `ticketing` contract crate, independent from the `shade` hub's `EventError` (codes 120–139), which covers the hub's own built-in ticketing entry points.

| Error | Code | Trigger | Corrective action |
|---|---:|---|---|
| `EventNotFound` | 1 | The event ID does not exist. | Confirm the ID from the event-creation call. |
| `TicketNotFound` | 2 | The ticket ID does not exist. | Confirm the ID from the ticket-purchase call. |
| `NotAuthorized` | 3 | Caller lacks the required role for the operation. | Call with the correct signer for this operation. |
| `EventAtCapacity` | 4 | Ticket capacity has been reached. | No further tickets can be issued for this event. |
| `DuplicateQRHash` | 5 | The supplied QR hash collides with an existing one. | Generate a new, unique QR hash. |
| `AlreadyCheckedIn` | 6 | The holder has already checked in for this event. | No further check-in action needed. |
| `TicketAlreadyCheckedIn` | 7 | This specific ticket has already been checked in. | No further check-in action needed. |
| `InvalidTimeRange` | 8 | A supplied time range is invalid (e.g. end before start). | Correct the time range. |
| `TierNotFound` | 9 | The referenced pricing tier does not exist. | Confirm the tier index against the event's configured tiers. |
| `TierAtCapacity` | 10 | The tier has reached its own capacity limit. | Select a different tier, or wait for capacity to free up. |
| `InvalidTierSupply` | 11 | The tier's configured supply is invalid. | Supply a valid, positive tier capacity. |
| `InvalidTierPrice` | 12 | The tier's configured price is invalid. | Supply a valid, positive tier price. |
| `TierEventMismatch` | 13 | The tier does not belong to the referenced event. | Confirm the tier ID belongs to the event you're operating on. |
| `InvalidRoyaltyBps` | 14 | Royalty basis points exceed the accepted maximum. | Supply a royalty within the valid basis-point range. |
| `InvalidResalePrice` | 15 | Resale price falls outside the enforced bound. | Set a resale price within the enforced band. |
| `ResaleNotConfigured` | 16 | Resale was attempted on an event with no resale configuration. | The event owner must configure resale terms first. |
| `SameHolder` | 17 | A resale or transfer targets the ticket's current holder. | Choose a different recipient address. |
| `EventCancelled` | 18 | The event was cancelled. | No further ticket operations are possible on a cancelled event. |
| `TicketAlreadyRefunded` | 19 | This ticket was already refunded. | No further action needed. |
| `NotAtCapacity` | 20 | An operation requiring a full/at-capacity event was attempted on one that is not. | Wait until the event reaches capacity, or check its state first. |
| `AlreadyOnWaitlist` | 21 | The caller is already on the event's waitlist. | No further waitlist action needed. |

## `ticketing_factory` (`contracts/ticketing_factory/src/errors.rs`)

| Error | Code | Trigger | Corrective action |
|---|---:|---|---|
| `NotInitialized` | 1 | An operation was attempted before `initialize` ran. | Initialize the factory before deploying ticketing instances. |
| `AlreadyInitialized` | 2 | `initialize` was already called on the factory. | No action needed. |
| `NotAuthorized` | 3 | Caller is not the factory admin. | Call with the factory's configured admin address. |
| `WasmHashNotSet` | 4 | Deployment attempted before `set_ticketing_wasm_hash` was called. | Admin must call `set_ticketing_wasm_hash` first. |
| `EventRefNotFound` | 5 | The referenced event does not exist in the factory's own records. | Confirm the event reference was recorded during deployment. |

## `subscription` (`contracts/subscription/src/errors.rs`)

The standalone `subscription` contract crate, independent from the `shade` hub's own built-in subscription entry points (which use `ContractError` codes 21–26, 41).

| Error | Code | Trigger | Corrective action |
|---|---:|---|---|
| `AlreadyInitialized` | 1 | `initialize` was already called. | No action needed. |
| `NotInitialized` | 2 | An operation was attempted before `initialize` ran. | Initialize the contract before use. |
| `NotAuthorized` | 3 | Caller lacks the required role. | Call with the correct signer for this operation. |
| `InvalidAmount` | 4 | A supplied amount is zero or negative. | Supply a positive amount. |
| `InvalidInterval` | 5 | The billing interval is zero or otherwise invalid. | Supply a positive interval in seconds. |
| `PlanNotFound` | 6 | The plan ID does not exist. | Confirm the ID from plan creation. |
| `PlanNotActive` | 7 | The plan was deactivated. | Cannot subscribe to a deactivated plan. |
| `SubscriptionNotFound` | 8 | The subscription ID does not exist. | Confirm the ID from the subscribe call. |
| `SubscriptionNotActive` | 9 | The subscription is not active (cancelled or terminated). | No further charges possible. |
| `ChargeTooEarly` | 10 | The billing interval has not yet elapsed. | Wait until the interval elapses. |
| `InsufficientAllowance` | 11 | The customer's SEP-41 allowance to this contract is insufficient. | Increase the token allowance before charging. |
| `TokenNotAccepted` | 12 | The plan's token is not globally accepted. | Confirm the token was whitelisted before plan creation. |
| `SubscriptionTerminated` | 13 | An operation was attempted on a terminated subscription. | No further action possible; the subscription is permanently ended. |
| `GraceNotExpired` | 14 | An operation requiring an expired grace period was attempted too early. | Wait for the grace period to elapse. |
| `NothingToRefund` | 15 | A refund was attempted with no refundable balance. | No refund is possible; confirm the subscription's payment history first. |

## `crowdfund` (`contracts/crowdfund/src/errors.rs`)

Not one of the seven crates named in the original tracking issue, but a real error enum in the workspace — included here for completeness per the coverage requirement.

| Error | Code | Trigger | Corrective action |
|---|---:|---|---|
| `AlreadyInitialized` | 1 | `initialize` was already called. | No action needed. |
| `NotInitialized` | 2 | An operation was attempted before `initialize` ran. | Initialize the contract before use. |
| `InvalidGoal` | 3 | The funding goal is not positive. | Supply a positive goal amount. |
| `InvalidDeadline` | 4 | The deadline is not in the future. | Supply a future deadline. |
| `InvalidAmount` | 5 | A pledge or withdrawal amount is invalid. | Supply a positive, valid amount. |
| `CampaignEnded` | 6 | The campaign's deadline has passed. | No further pledges accepted after the deadline. |
| `CampaignNotEnded` | 7 | An operation requiring a closed campaign was attempted before its deadline. | Wait until the deadline passes. |
| `GoalNotReached` | 8 | Organizer withdrawal attempted when the goal was not reached. | Withdrawal is only possible once the goal is met. |
| `GoalReached` | 9 | A refund was attempted after the goal was reached. | Refunds are only available when the goal was not reached. |
| `NoPledge` | 10 | The contributor has no recorded pledge to refund. | Confirm a pledge was recorded for this address. |
| `AlreadyExecuted` | 11 | Funds have already been withdrawn by the organizer. | No further withdrawal is possible. |
| `AlreadyFulfilled` | 12 | The backer's reward was already marked fulfilled. | No further action needed. |
| `PledgeBelowTierMinimum` | 13 | The contributor's total pledge is below the selected tier's minimum. | Increase the pledge or select a lower tier. |
| `InvalidTier` | 14 | The supplied tier index does not exist. | Confirm the tier index against configured tiers. |
| `MilestonesNotSet` | 15 | An operation requiring milestones was attempted with none configured. | Configure milestones before using milestone-mode operations. |
| `MilestoneAlreadyReleased` | 16 | This milestone was already released. | No further release possible for this milestone. |
| `MilestoneNotUnlocked` | 17 | This milestone has not yet been unlocked by the organizer. | Wait for the organizer to unlock the milestone. |
| `InvalidMilestonePercentages` | 18 | Milestone percentages are zero or do not sum to exactly 10,000 bps (100%). | Correct the percentages to sum to exactly 10,000 bps. |
| `MilestonesActive` | 19 | `execute_campaign` was called while the campaign is in milestone mode. | Use `release_milestone` instead of `execute_campaign`. |
| `MilestoneNotApproved` | 20 | Milestone release lacks a strict majority approval from backers. | Wait for more backer votes before releasing. |
| `NotBacker` | 21 | Only contributors with a recorded pledge can vote on milestone release. | Only backers may cast a milestone vote. |
| `MilestoneVoteAlreadyCast` | 22 | A backer can only vote once per milestone. | No further vote possible from this backer on this milestone. |
| `ShadeGatewayNotSet` | 23 | The Shade payment gateway address has not been configured. | Configure the gateway address before accepting Shade-routed payments. |
| `MerchantAccountNotSet` | 24 | The merchant account address has not been configured. | Configure the merchant account before withdrawal. |
| `RefundAlreadyProcessed` | 25 | The batch refund has already been processed. | No further action needed. |
| `InsufficientMatchingPool` | 26 | The sponsor matching pool cannot satisfy a match request. | Reduce the requested match, or wait for the pool to be topped up. |
| `CommentTooLong` | 27 | The pledge comment exceeds the configured maximum length. | Shorten the comment. |
| `AffiliateNotRegistered` | 28 | The caller is not registered as an affiliate for this campaign. | Register as an affiliate before referring contributions. |
| `ReferralCodeAlreadyTaken` | 29 | This referral code has already been registered by another affiliate. | Choose a different referral code. |
| `InvalidCommissionBps` | 30 | Commission basis points fall outside 0–10,000. | Supply a commission between 0 and 10,000 bps. |
| `ReferralCodeNotFound` | 31 | No affiliate is registered with the supplied referral code. | Confirm the referral code against registered affiliates. |
| `ReferralAlreadyUsed` | 32 | This contributor already used a referral code for this campaign. | Only one referral code may be applied per contributor per campaign. |
| `NotAuthorized` | 33 | Caller is not authorized for this privileged view (organizer only). | Only the campaign organizer may perform this operation. |
| `BadgeAlreadyAwarded` | 34 | The backer already holds this badge. | No further action needed. |
| `BadgeNotEligible` | 35 | The backer does not meet this badge's on-chain eligibility rules. | Meet the eligibility criteria before claiming. |
| `BadgeConfigNotSet` | 36 | Badge eligibility thresholds have not been configured by the organizer. | Organizer must configure badge thresholds first. |
| `GuardiansNotSet` | 37 | No guardian set has been configured for this campaign. | Configure guardians before initiating recovery. |
| `DuplicateGuardian` | 38 | The same address appears twice in the supplied guardian list. | Remove duplicate addresses from the guardian list. |
| `InvalidThreshold` | 39 | Threshold is zero or greater than the number of guardians. | Supply a threshold between 1 and the guardian count. |
| `RecoveryAlreadyPending` | 40 | A recovery is already pending; must be cancelled before starting another. | Cancel the pending recovery first. |
| `NoPendingRecovery` | 41 | No recovery is currently pending. | Initiate a recovery before approving one. |
| `NotGuardian` | 42 | The caller is not in the registered guardian set. | Only registered guardians may approve recovery. |
| `AlreadyApprovedRecovery` | 43 | This guardian already approved the pending recovery. | No further action needed. |
| `KYCRequired` | 44 | The campaign requires KYC and the contributor is not verified. | Complete KYC verification before contributing. |

## `crowdfund_factory` (`contracts/crowdfund_factory/src/errors.rs`)

Also not one of the seven crates named in the original tracking issue, included here for full workspace coverage.

| Error | Code | Trigger | Corrective action |
|---|---:|---|---|
| `NotInitialized` | 1 | An operation was attempted before `initialize` ran. | Initialize the factory before deploying campaigns. |
| `AlreadyInitialized` | 2 | `initialize` was already called on the factory. | No action needed. |
| `WasmHashNotSet` | 3 | Deployment attempted before `set_crowdfund_wasm_hash` was called. | Call `set_crowdfund_wasm_hash` first. |
| `CampaignNotFound` | 4 | The referenced campaign does not exist in the factory's records. | Confirm the campaign was deployed through this factory. |
| `GovernanceNotInitialized` | 5 | A governed action was attempted before `init_governance` ran. | Initialize governance first. |
| `NotReviewer` | 6 | Caller is neither the governance admin nor a granted reviewer. | Only the admin or a granted reviewer may perform this action. |
| `InvalidGoal` | 7 | A proposed campaign's goal is not positive. | Supply a positive goal amount. |
| `InvalidDeadline` | 8 | A proposed campaign's deadline is not in the future. | Supply a future deadline. |
| `ProposalNotFound` | 9 | The referenced governance proposal does not exist. | Confirm the proposal ID. |
| `ProposalNotPending` | 10 | The proposal is not in `Pending` status. | Check the proposal's status before acting on it. |
| `ProposalNotApproved` | 11 | An action requiring approval was attempted on an unapproved proposal. | Wait for the proposal to be approved first. |
| `DaoNotInitialized` | 12 | A DAO-governed action was attempted before `init_dao` ran. | Initialize the DAO first. |
| `DaoAlreadyInitialized` | 13 | `init_dao` was already called. | No action needed. |
| `NotDaoAdmin` | 14 | Caller is not the DAO admin. | Call with the DAO admin address. |
| `AlreadyDaoMember` | 15 | The address is already a DAO member. | No further action needed. |
| `NotDaoMember` | 16 | Caller is not a registered DAO member. | Only DAO members may perform this action. |
| `InvalidQuorum` | 17 | Quorum is zero or exceeds 10,000 basis points. | Supply a quorum between 1 and 10,000 bps. |
| `InvalidVotingPeriod` | 18 | The voting period is invalid. | Supply a positive voting period. |
| `DaoProposalNotFound` | 19 | The referenced DAO proposal does not exist. | Confirm the proposal ID. |
| `VotingClosed` | 20 | The proposal's voting window has closed, or it is no longer in `Voting` status. | No further votes accepted. |
| `VotingNotClosed` | 21 | Finalization was attempted before the voting window closed. | Wait until the voting window closes. |
| `AlreadyVoted` | 22 | Caller already voted on this proposal. | Each member may vote once per proposal. |
| `DaoProposalAlreadyExecuted` | 23 | The proposal was already executed. | No further action possible. |

> **Warning:** `set_crowdfund_wasm_hash` on this factory performs **no** `require_auth` and no admin check — any address can repoint it. See [Upgradeability: hub upgrades versus factory-deployed WASM](../architecture/upgradeability.md#hub-upgrades-versus-factory-deployed-wasm) for the full explanation of this known gap.

## Reading an error from a failed transaction

### Soroban CLI

When a `stellar contract invoke` call panics with a `#[contracterror]` variant, the CLI prints the error inline in its failure output, in the form `HostError: Error(Contract, #<code>)`. For example, calling `pay_invoice` against an invoice paid in a token the merchant does not personally accept (only globally whitelisted) fails like this against a local/testnet deployment:

```bash
stellar contract invoke \
  --id $SHADE_ID \
  --source payer \
  --network testnet \
  -- pay_invoice \
  --payer $(stellar keys address payer) \
  --invoice_id 1
```

```text
❌ error: transaction simulation failed: HostError: Error(Contract, #41)
```

The `#41` is the raw discriminant. Match it against the **contract that raised it** — `$SHADE_ID` here is the `shade` contract, so `41` resolves to `ContractError::TokenNotAcceptedByMerchant` in the table above, not to any other contract's code `41`.

> **Note:** The exact wording of CLI failure output (`HostError: Error(Contract, #N)` vs. a differently formatted panic message) depends on the installed `stellar-cli` version. The example above reflects the CLI version pinned in [Prerequisites](../getting-started/prerequisites.md) (23.x) at the time of writing; always confirm against your installed `stellar --version` if the format looks different.

### SDK / RPC client errors

A Soroban RPC client (for example a TypeScript or Rust SDK built against `soroban-rpc`) distinguishes several failure layers, and only the last one maps to this page's tables:

| Layer | What it means | Where the contract error code appears |
|---|---|---|
| Transport/RPC failure | The RPC endpoint was unreachable, rate-limited, or returned malformed JSON. | Not applicable — no contract code was ever reached. |
| Transaction simulation/submission failure | The transaction failed before or during execution for a reason outside contract logic (insufficient fee, bad sequence number, expired ledger footprint). | Not applicable — the failure is at the Soroban/Stellar transaction layer, not a `#[contracterror]` variant. |
| Contract error (host trap) | The invoked contract function panicked via `panic_with_error!`. | Present in the transaction result's `resultXdr`, decodable to `Error(Contract, #<code>)` — this is what this page documents. |
| Application-level validation error | A client-side check (e.g. a webhook consumer validating a payload) rejects data before or after the on-chain call. | Not a contract error at all; this is validation logic outside Soroban, documented per-integration rather than here. |

Regardless of SDK, the contract-error code lives in the transaction result and is the same numeric discriminant the CLI prints. Decode it against the specific contract ID that raised it, then look it up in this page.

## Troubleshooting shortlist

A practical shortlist of failures most likely during common flows — not a claim that these are globally the most frequent, just the ones worth checking first for these operations.

| Symptom | Likely error / cause | Fix |
|---|---|---|
| `pay_invoice` fails during merchant onboarding smoke-testing | `TokenNotAcceptedByMerchant` (41) — merchant hasn't called `set_merchant_accepted_tokens`, or `TokenNotAccepted` (12) — token was never globally whitelisted | Check `is_token_accepted_for_merchant`; have admin run `add_accepted_token` and/or merchant run `set_merchant_accepted_tokens` |
| `create_invoice` fails right after registration | `MerchantNotActive` (32) | Admin must call `set_merchant_status(admin, merchant_id, true)` |
| `pay_invoice` fails with a balance-looking error | `InsufficientBalance` (30) | Fund or approve the payer's token balance |
| Invoice payment rejected without an obvious reason | `InvoiceExpired` (27) | Check `get_invoice`'s `expires_at` against the current ledger time |
| `create_invoice_signed` always fails | `MerchantKeyNotFound` (11) — merchant never called `set_merchant_key` | Register a key first; see [Signed invoices](../security/signatures.md) |
| Subscription won't charge | `ChargeTooEarly` (26) or `SubscriptionNotActive` (25) | Check the plan interval and subscription status via `get_subscription` |
| `purchase_ticket` fails at a popular event | `EventSoldOut` (121) | Check `get_event` capacity vs. `get_event_tickets` |
| Admin operation rejected unexpectedly | `NotAuthorized` (1, or contract-specific equivalent) | Confirm you're signing with the actual current admin (`get_admin`), not a stale or assumed address |
| Upgrade proposal won't finalize | `VotingStillOpen` (106) or `ProposalNotActive` (104) | Check `get_upgrade_proposal`'s `voting_ends_at` and status |

## Append-only numbering convention

New error variants must be **appended** with a new, unused number in the enum's range — never inserted with a renumbered existing variant, and never reusing a retired number for a different meaning. This mirrors the `#[repr(u32)]` integer-enum rule documented in [Upgradeability: integer enums](../architecture/upgradeability.md#integer-enums-the-number-is-the-identity):

- A stored or previously-observed error code is effectively part of the protocol's wire format — client code, indexers, and alerting rules match on it.
- Renumbering `TokenNotAccepted` from 12 to some other value would silently break every integrator's existing `if code == 12` check without any compile-time signal.
- Removing a variant and later adding a new one with the same number changes its meaning for anyone who cached the old mapping.

The Rust compiler does **not** enforce this — nothing prevents a future PR from reusing a freed number or reordering `= N` discriminants. It is a required convention for this workspace, not a compiler guarantee, and should be checked in review the same way the [upgrade checklist](../architecture/upgradeability.md#checklist-before-shipping-an-upgrade) checks storage-key stability.

When a domain enum approaches its 50-variant cap (see the note in [Upgradeability](../architecture/upgradeability.md#key-enums-the-variant-name-is-the-identity)), start a new domain enum with a fresh numeric range rather than renumbering the existing one.

## Related pages

- [Upgradeability, WASM hash management, and versioning](../architecture/upgradeability.md) — the append-only convention for `#[repr(u32)]` enums that this page's numbering convention mirrors.
- [Signed invoices](../security/signatures.md) — detail on `MerchantKeyNotFound` and `NonceAlreadyUsed`.
- [Merchant Onboarding Guide](../guides/merchant-onboarding.md) — the errors most likely to appear during onboarding.

← [Back to reference](README.md)
