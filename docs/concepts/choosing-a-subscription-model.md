# Choosing a subscription model

Shade ships two independent subscription systems: a simple one built into the hub, and the standalone `subscription` contract. This page compares them and tells you which to use. It is for integrators deciding where to bill recurring payments.

## Why it exists

Both systems expose plans, subscribing, and charging, so the names look interchangeable. They are not. They share no code, types, storage, or runtime calls, and a [merchant](../glossary.md#merchant) cannot mix them for one billing relationship. Pick one per relationship before you build.

## Decision summary

- Use the **hub** when subscription revenue must flow through Shade's merchant registry, platform fee split, merchant account settlement, and analytics.
- Use the **standalone contract** when you need grace periods, prorated refunds, upgrades and downgrades, trial periods, or batch billing, and you do not need the hub's fee routing or analytics.

## Feature comparison

| Capability | Hub (`shade`) | Standalone (`subscription`) |
|---|---|---|
| Entry points | `create_subscription_plan`, `subscribe`, `charge_subscription`, `cancel_subscription`, `deactivate_plan` | `create_plan`, `subscribe`, `subscribe_with_token`, `charge`, `process_charge`, `process_billing_cycle`, `cancel_subscription`, `cancel_with_prorated_refund`, and more |
| Who can create a plan | A registered [merchant](../glossary.md#merchant) (`MerchantNotFound` otherwise). Token must be on the hub whitelist. | Any address that authorizes. Token must be on the contract's own `AcceptedTokens`, set by its admin. |
| Plan management | Create and deactivate only. `amount`, `interval`, and `token` are fixed at creation. | `update_plan_amount`, `update_plan_interval`, `set_plan_grace_period`, `deactivate_plan` |
| Billing authorization | The customer approves the **hub contract** as a token spender directly on the token. There is no authorize or revoke function. | `authorize_billing(customer, sub_id, cycles)`, `get_billing_allowance`, `revoke_billing_authorization`, with the **subscription contract** as spender. See [billing authorization](./subscription-billing-authorization.md). |
| Who triggers a charge | Anyone. `charge_subscription` takes no caller and no auth. | Anyone, for `charge`, `process_charge`, and `process_billing_cycle`. |
| First charge | Available immediately (`last_charged` starts at `0`). | Gated by `Plan.trial_period` from the subscription's `created_at`. |
| Failed charge | Panics. The transaction reverts and the subscription stays `Active`. | `charge` panics with `InsufficientAllowance`. `process_charge` and `process_billing_cycle` move the subscription to `PastDue` instead. |
| Grace periods | None. Statuses are `Active` and `Cancelled` only. | Yes. `Active`, `Cancelled`, `PastDue`, `Terminated`. See [grace periods](./subscription-grace-periods.md). |
| Prorated refunds | None. | `cancel_with_prorated_refund` and `quote_prorated_refund`. Floor division on `amount * remaining / interval`. |
| Upgrade and downgrade | None. | `upgrade_subscription` (immediate, prorated) and `downgrade_subscription` (deferred to the next charge). |
| Batch processing | None. One `charge_subscription` call per subscription. | `process_billing_cycle(sub_ids)` returns one `ChargeOutcome` per ID and skips terminal entries instead of aborting. |
| Multi-token | Plan token only. | Per-subscription `preferred_token` via `subscribe_with_token` and `set_preferred_token`. |
| Platform fee | Yes. `route_from_allowance` splits each charge by the effective fee in basis points. Fee is taken out of `plan.amount`. | None. 100% of `plan.amount` goes to the recipient. |
| Settlement destination | The merchant's account contract (`get_merchant_account`) and the platform account. | `plan.creator` if set, otherwise `plan.merchant`. |
| Analytics and history | Counted in merchant and [token analytics](./token-analytics.md), and recorded in the customer's transaction log as `SubscriptionCharge`. | Not recorded in hub analytics or history. |
| Cancellation | Customer or plan merchant, from `Active` only. | Customer or plan merchant, from `Active` or `PastDue`. |
| Search | `search_subscription_plans` and `search_subscriptions` on the hub. | `get_plan`, `get_plan_count`, `get_subscription`. No filtered search. |
| Upgradeable | Yes, through the hub upgrade flow. | No. Immutable once deployed. |

## Data types

| Concept | Hub | Standalone |
|---|---|---|
| Plan | `SubscriptionPlan { id, merchant_id, merchant, description, token, amount, interval, active }` in [`contracts/shade/src/types.rs#L491`](../../contracts/shade/src/types.rs#L491) | `Plan` in [`contracts/subscription/src/types.rs#L34`](../../contracts/subscription/src/types.rs#L34), adds `created_at`, `grace_period`, `creator`, `trial_period` and has no `merchant_id` |
| Subscription | `Subscription { id, plan_id, customer, merchant_id, status, date_created, last_charged }` in [`contracts/shade/src/types.rs#L511`](../../contracts/shade/src/types.rs#L511) | `Subscription` in [`contracts/subscription/src/types.rs#L56`](../../contracts/subscription/src/types.rs#L56), adds `past_due_since`, `pending_downgrade_plan_id`, `preferred_token`, and names the timestamp `created_at` |
| Status | `SubscriptionStatus { Active = 0, Cancelled = 1 }` | `SubscriptionStatus { Active = 0, Cancelled = 1, PastDue = 2, Terminated = 3 }` |
| Merchant identity | Numeric `merchant_id` from the registry | Plain `Address` |
| IDs | Sequential `u64`, per contract | Sequential `u64`, per contract. The same number can mean different things in each system. |

> **Warning:** Both systems number plans and subscriptions from 1 independently. Always store the contract address with the ID.

## Events

| Moment | Hub | Standalone |
|---|---|---|
| Plan created | `SubscriptionPlanCreatedEvent` | None |
| Subscribed | `SubscribedEvent` | None |
| Charged | `SubscriptionChargedEvent` (includes `amount`, `fee`, `token`), plus `PlatformFeeRoutedEvent` | `SubRenewed` (`subscription_id`, `plan_id`, `customer`, `timestamp`). No amount field. |
| Cancelled | `SubscriptionCancelledEvent` | `SubExpired`, also used for termination and prorated-refund cancellation |
| Plan deactivated | `PlanDeactivatedEvent` | None |
| Terminated after grace | Not applicable | `SubExpired` |

An indexer must treat the two contracts as separate sources. See [Indexing events](../guides/indexing-events.md).

## Recommendations by use case

| Use case | Choose | Why |
|---|---|---|
| Simple fixed billing, paid to a registered merchant, with Shade fees | Hub | Fee split, merchant account settlement, and analytics come for free. A failed charge just reverts, and your keeper retries. |
| SaaS billing with dunning (retry, grace, then terminate) | Standalone | `PastDue`, `grace_period`, `enforce_grace`, and `process_billing_cycle` implement dunning on-chain. |
| Tiered SaaS with plan changes | Standalone | The only system with `upgrade_subscription` and `downgrade_subscription`. |
| Free trial before the first charge | Standalone | The hub charges immediately. |
| Refund unused time on cancellation | Standalone | The hub has no refund path. |
| Usage-adjacent billing (variable amount per period) | Neither fully | Both charge a fixed `plan.amount`. The standalone contract allows `update_plan_amount` between cycles, which lets a keeper adjust a plan, but that changes the price for every subscriber of the plan. Bill true usage through [invoices](./invoice-lifecycle.md). |
| Customer-selectable payment token | Standalone | `subscribe_with_token` and `set_preferred_token`. |
| Revenue must appear in Shade dashboards and token share | Hub | Only hub charges are recorded. |

> **Note:** If you choose the standalone contract and still want Shade fees, you must collect them yourself. It never calls the hub.

## Migration

There is no migration function in either direction, and no shared state. Moving a merchant from one model to the other means starting fresh subscriptions.

Hub to standalone:

1. Deploy or locate a standalone `subscription` contract, and ask its admin to `add_accepted_token` for your token.
2. Create an equivalent plan with `create_plan`. Set `trial_period` so the first charge does not double-bill a customer who was just charged on the hub, and set `set_plan_grace_period` if you want dunning. `trial_period` is plan-wide and measured from each subscription's `created_at`, so staggered customers need separate plans or an off-chain delay before calling `charge`.
3. Ask each customer to call `subscribe` and `authorize_billing`, which approves a different spender than the hub allowance.
4. Cancel the hub subscription with `cancel_subscription` (customer or merchant) so `charge_subscription` can no longer be called. The customer should also set the hub contract's token allowance to `0` directly on the token, because the hub has no revoke function.

Standalone to hub:

1. Register the merchant on the hub and create a plan with `create_subscription_plan`. The plan `amount` must cover the platform fee.
2. Have each customer `subscribe` on the hub and approve the hub contract as a token spender for the expected total.
3. Cancel the standalone subscription with `cancel_subscription`, or `cancel_with_prorated_refund` to return unused time, and call `revoke_billing_authorization`.

Differences to plan for:

| Item | Effect |
|---|---|
| Billing history | `last_charged`, `past_due_since`, and event history do not carry over. |
| Fees | Moving to the hub introduces a platform fee. Moving off it removes the fee and the analytics entry. |
| Customer consent | Every customer must act on-chain, since the spender and the contract change. |
| Plan and subscription IDs | New IDs are issued. Update your database keys. |

## Maintainer intent

The code and existing docs establish that the two systems are parallel, with no delegation or wrapping between them, and no factory for the standalone contract. They do not record a decision about the long-term relationship, such as converging them, deprecating one, or having the hub call the standalone contract.

> **Note:** **Open question for maintainers:** Is the hub subscription component intended to stay a minimal fixed-billing feature, gain the standalone contract's features, or be replaced by a hub integration with the standalone contract? Until this is answered in an issue or ADR, build against the behavior documented here and do not assume convergence.

## Related pages

- [Subscriptions and recurring billing](./subscriptions.md)
- [Subscription contract reference](../contracts/subscription.md)
- [Subscription billing authorization and allowances](./subscription-billing-authorization.md)
- [Subscription grace periods and past-due handling](./subscription-grace-periods.md)
- [Billing keeper operator guide](../operations/billing-keeper.md)
- [Token analytics and market share metrics](./token-analytics.md)
