# Subscription billing authorization and allowances

The standalone `subscription` contract (`contracts/subscription/`) never custodies a customer's tokens directly — every recurring charge is pulled through a standard token allowance the customer grants to the contract. This page documents how a customer authorizes billing, how that allowance is consumed cycle by cycle, and what happens when it runs out or the plan's price changes.

> **Note:** This page covers the standalone `subscription` crate, not the Shade hub's own subscription feature. See [Subscriptions and recurring billing](./subscriptions.md) for the hub's `charge_subscription` flow, and [Subscription contract reference](../contracts/subscription.md) for the full function/type/error listing this page builds on.

## Why it exists

Recurring billing needs a way for a customer to pre-authorize many future charges without signing a transaction for every one. Rather than invent a bespoke payment-routing structure (as the Shade hub does with `platform_fee::route_from_allowance`), this contract reuses the token's own SEP-41 `approve`/`allowance`/`transfer_from` interface: authorizing billing *is* approving the subscription contract as a spender. There is no separate "contract-level allowance" ledger — `get_billing_allowance` and every internal sufficiency check read the token's live allowance directly.

## How it works

### `authorize_billing(customer, sub_id, cycles)`

[`contracts/subscription/src/lib.rs`](../../contracts/subscription/src/lib.rs#L390-L407)

```rust
pub fn authorize_billing(env: Env, customer: Address, sub_id: u64, cycles: u32)
```

- **Auth:** `customer.require_auth()`. Panics `NotAuthorized` if `customer` isn't the subscription's own customer, and `SubscriptionNotActive` if the subscription isn't currently `Active`.
- **Allowance amount:** `allowance_amount = plan.amount.saturating_mul(cycles as i128)` — the plan's per-cycle charge times the number of cycles you're authorizing. `saturating_mul` means a pathologically large `cycles` clamps to `i128::MAX` instead of overflowing, rather than panicking.
- **Allowance expiration:** `ledger_expiry = current_ledger_sequence + 17_280 * cycles`. `17_280` is a fixed approximation of one day's worth of ledger closes (5-second average close time), **not derived from `plan.interval`**. For a monthly plan this comfortably covers `cycles` months; for a plan with a much shorter or longer interval than a day, the expiration may lapse well before or persist well after the number of cycles it was meant to cover. Size `cycles` with your plan's actual interval in mind, not just "how many charges do I want to cover."
- **What it calls:** `token::TokenClient::approve(&customer, &spender, &allowance_amount, &ledger_expiry)`, where `spender` is the subscription contract's own address (`env.current_contract_address()`). This is an ordinary token `approve` — the exact same call any SEP-41 token exposes to any spender, not a subscription-specific entry point.
- **Billing token used:** `sub.preferred_token.unwrap_or(plan.token)` — if the customer set a preferred token via `subscribe_with_token`/`set_preferred_token`, authorization (and every later charge) acts on that token instead of the plan's default.

Calling `authorize_billing` again **replaces** the allowance rather than adding to it — `approve` sets an absolute amount and expiration, it does not accumulate. Re-authorizing for `cycles = 6` after a previous `cycles = 12` authorization (even if none of the original 12 have been consumed) leaves the allowance at exactly 6 cycles' worth, not 18.

### `get_billing_allowance(customer, sub_id)`

[`contracts/subscription/src/lib.rs`](../../contracts/subscription/src/lib.rs#L409-L419)

```rust
pub fn get_billing_allowance(env: Env, customer: Address, sub_id: u64) -> i128
```

- **Auth:** none — this is a read-only call, anyone can query it.
- **What it returns:** the live token allowance from `customer` to the contract, on the subscription's billing token, via `token_client.allowance(&customer, &spender)`. It is not a cached or contract-tracked figure; it reflects exactly what the token contract reports right now, including any reduction from charges already pulled and any expiration that has already passed (an expired allowance reads back as `0`, same as the token semantics).
- **Use it to verify sufficiency before attempting a charge** — a keeper or UI can call this ahead of `process_charge`/`charge` to predict whether the attempt will succeed, without needing to simulate the charge itself.

### `revoke_billing_authorization(customer, sub_id)`

[`contracts/subscription/src/lib.rs`](../../contracts/subscription/src/lib.rs#L421-L436)

```rust
pub fn revoke_billing_authorization(env: Env, customer: Address, sub_id: u64)
```

- **Auth:** `customer.require_auth()`. Panics `NotAuthorized` if `customer` isn't the subscription's own customer.
- **Effect:** sets the token allowance to `0` via `approve(&customer, &spender, &0, &current_sequence)` — it does **not** change `Subscription.status`. The subscription record stays `Active` (or whatever it currently is); only future charge attempts are affected, since they'll now find an insufficient allowance.
- This is the tool for "stop billing me without cancelling my plan outright." Note that `cancel_subscription`, when called by the customer, already zeroes the allowance as part of the same transaction (see [Billing authorization and the contract-level allowance](#billing-authorization-and-the-contract-level-allowance) below) — calling `revoke_billing_authorization` after a customer-initiated cancel is a harmless no-op.

## Billing authorization and the contract-level allowance

There is only one allowance in play: the token's own. "Authorizing billing" and "approving the token" are the same operation under the hood — `authorize_billing` is a thin wrapper around `token::approve` that computes the amount and expiry for you from the plan and cycle count. What the customer actually signs is a standard Soroban authorization for an `approve` invocation on the billing token contract, naming the subscription contract as spender, for `plan.amount * cycles`, expiring at a given ledger sequence. Nothing about the signed payload is subscription-specific or human-readable as "12 months of Pro Plan" — a wallet will present it as a token approval, the same way it would for any other spender.

Each successful charge consumes the allowance automatically: `charge()`, `process_charge()`, and `process_billing_cycle()` all pull funds via `token_client.transfer_from(&spender, &customer, &recipient, &plan.amount)`, and `transfer_from` is what decrements the live allowance by `plan.amount` under standard SEP-41 semantics. The subscription contract itself keeps no running "cycles remaining" counter — `get_billing_allowance` after each charge simply reflects `previous_allowance - plan.amount`, computed by the token contract, not by this contract.

## What happens on insufficient allowance

When a charge is attempted and the live allowance is below `plan.amount`:

- **Strict `charge(sub_id)`** panics `InsufficientAllowance` immediately. Nothing changes — the subscription stays in its prior status.
- **Forgiving `process_charge(sub_id)` / `process_billing_cycle(sub_ids)`** never panic for this reason. Instead they drive the grace-period state machine: if the plan has `grace_period > 0`, the subscription moves to `PastDue` and the call returns `ChargeOutcome::EnteredGrace`; if `grace_period == 0`, the subscription is terminated immediately and the call returns `ChargeOutcome::Terminated`.

See [Subscription grace periods and past-due handling](./subscription-grace-periods.md) for the full `PastDue`/`Terminated` lifecycle, recovery rules, and the events each transition does (or doesn't) emit.

## Effect of `update_plan_amount` on existing allowances

`update_plan_amount(merchant, plan_id, new_amount)` ([`contracts/subscription/src/lib.rs`](../../contracts/subscription/src/lib.rs#L178-L191)) only writes `Plan.amount`. It touches no `Subscription` record and makes no token call — every allowance any customer has already granted via `authorize_billing` is completely unaffected by this call, for better or worse:

- **Price increase:** a customer who authorized `12 * old_amount` now has an allowance that, at `new_amount` per cycle, covers fewer than 12 charges. The next charge still succeeds as long as the live allowance is `>= new_amount`; the shortfall only surfaces once consumption catches up with the smaller effective cycle count, at which point the customer silently enters grace (or gets terminated, on a zero-grace plan) with no on-chain signal that the cause was a price change rather than inattention.
- **Price decrease:** the same allowance now covers more than 12 charges at the new, lower rate — harmless, just an under-used authorization.
- **Nothing re-authorizes automatically.** The contract has no notion of "re-prompt the customer" — that is purely an off-chain/UI responsibility.

> **Note:** Because `update_plan_amount` leaves `get_billing_allowance` unchanged while silently changing how many future charges that allowance actually covers, a UI should treat any price change as a trigger to re-check allowance coverage for every affected subscription, not just show the new price. Recommended flow: watch for the amount change (there is no dedicated event for it — poll `get_plan` or diff it against your own cached copy), compute `remaining_cycles_covered = get_billing_allowance(customer, sub_id) / new_amount`, and if that number looks low, prompt the customer to call `authorize_billing` again sized to the new `amount` before their next charge is due. Since `authorize_billing` replaces rather than adds to the allowance, prompt for the customer's **full** desired remaining cycle count, not just a top-up delta.

## Client example: subscribe and authorize 12 monthly cycles

This example uses the generated TypeScript bindings pattern described in [Generating and using contract bindings](../guides/contract-bindings.md), applied to the standalone `subscription` contract. It performs both calls (`subscribe`, then `authorize_billing`) in one user flow, since a freshly created subscription has no allowance until the customer explicitly grants one.

```typescript
import { Client } from "./bindings/subscription/src/index.js";

const client = new Client({
  contractId: "CSUBSCRIPTIONPLACEHOLDER00000000000000000000000000000000", // placeholder — your deployed subscription contract ID
  networkPassphrase: "Test SDF Network ; September 2015",                 // testnet passphrase; use the mainnet passphrase in production
  rpcUrl: "https://soroban-testnet.stellar.org",
});

const PLAN_ID = 1n;              // the merchant's monthly plan
const CYCLES_TO_AUTHORIZE = 12;  // u32 — one year of monthly billing

async function subscribeAndAuthorize(customerAddress: string, signTransaction: (xdr: string) => Promise<string>) {
  // Step 1: enroll in the plan. Returns the new subscription ID.
  const subscribeTx = await client.subscribe({
    customer: customerAddress,
    plan_id: PLAN_ID,
  });
  const subscribed = await subscribeTx.signAndSend({ signTransaction });
  const subId: bigint = subscribed.result; // u64, returned as bigint

  // Step 2: authorize the token allowance covering 12 monthly charges.
  // This is a plain token `approve` under the hood — the customer's wallet
  // will present it as such, sized from the plan's current amount.
  const authorizeTx = await client.authorize_billing({
    customer: customerAddress,
    sub_id: subId,
    cycles: CYCLES_TO_AUTHORIZE,
  });
  await authorizeTx.signAndSend({ signTransaction });

  // Step 3 (optional but recommended): confirm the allowance actually landed
  // before telling the customer they're all set.
  const { result: allowance } = await client.get_billing_allowance({
    customer: customerAddress,
    sub_id: subId,
  });
  console.log(`Subscription ${subId} authorized for ${allowance} base units`);

  return subId;
}
```

As with every write call in the generated client, `subscribe` and `authorize_billing` return an `AssembledTransaction` — call `.signAndSend({ signTransaction })` with your wallet's signing callback, never a hardcoded key (see [Using the generated client in an application](../guides/contract-bindings.md#using-the-generated-client-in-an-application)).

## Related pages

- [Subscription contract reference](../contracts/subscription.md) — full type, function, lifecycle, and error listing for this contract.
- [Subscription grace periods and past-due handling](./subscription-grace-periods.md) — what happens after an authorized allowance runs out.
- [Billing Keeper Operator Guide](../operations/billing-keeper.md) — running the off-chain process that calls `charge`/`process_charge`/`process_billing_cycle`.
- [Generating and using contract bindings](../guides/contract-bindings.md) — the TypeScript client pattern used above.
- [Subscriptions and recurring billing](./subscriptions.md) — the Shade hub's own, separate subscription feature.
