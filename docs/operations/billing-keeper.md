# Billing Keeper Operator Guide

How to run the off-chain process ("keeper") that drives recurring billing for the standalone `subscription` contract (`contracts/subscription/`). Nothing in this contract charges itself — every renewal, grace-period entry, recovery, and termination only happens when something outside the contract calls `charge`, `process_charge`, or `process_billing_cycle`. This guide is for whoever operates that something.

> **Note:** This guide covers the standalone `subscription` crate's keeper requirements. The Shade hub's own subscription feature (`charge_subscription`) has no grace period, no batch entry point, and no `ChargeOutcome` return value — see [Subscriptions and recurring billing](../concepts/subscriptions.md) if that's the contract you're integrating with instead.

## Charge entry points

[`contracts/subscription/src/lib.rs`](../../contracts/subscription/src/lib.rs)

| Function | Signature | Panics on terminal/ineligible state? | Returns |
|---|---|---|---|
| `charge` | `charge(sub_id: u64)` | **Yes** — panics `SubscriptionNotActive` on anything but `Active`, `ChargeTooEarly` if not yet due, `InsufficientAllowance` if the allowance can't cover `plan.amount`. | Nothing (unit) |
| `process_charge` | `process_charge(sub_id: u64) -> ChargeOutcome` | Panics only on `Cancelled`/`Terminated` (what `ChargeOutcome::Skipped` would otherwise mean) — [`contracts/subscription/src/lib.rs#L500-L506`](../../contracts/subscription/src/lib.rs#L500-L506). Everything else resolves into a return value instead of a panic. | `ChargeOutcome` |
| `process_billing_cycle` | `process_billing_cycle(sub_ids: Vec<u64>) -> Vec<ChargeOutcome>` | **No** — even a `Cancelled`/`Terminated` entry in the batch resolves to `ChargeOutcome::Skipped` rather than aborting the call. | `Vec<ChargeOutcome>`, same order as `sub_ids` |

- **`charge` is the strict, single-subscription primitive.** It has no grace-period awareness at all and no graceful outcome — a failed attempt panics the whole transaction. A keeper should use it only against plans it already knows have `grace_period == 0` and only when it's confident the charge is due and funded, since any panic reverts the entire call (no partial effect, no event, nothing to react to besides the failed transaction).
- **`process_charge` is the single-subscription forgiving primitive.** Prefer this over `charge` for almost every keeper use case — it drives the full `Active`/`PastDue`/`Terminated` state machine documented in [Subscription grace periods and past-due handling](../concepts/subscription-grace-periods.md) and tells you what happened via its return value, without ever reverting for an ordinary "not due yet" or "insufficient allowance" outcome. It still panics for an already-`Cancelled`/`Terminated` subscription — don't call it blind against an ID you haven't tracked the status of.
- **`process_billing_cycle` is the batch entry point** — hand it every `sub_id` you believe may be due in one call, and it tolerates dead entries. This is the function an actual production keeper should be calling on its regular sweep; `process_charge` is more useful for a one-off retry or a manual operator action against a single subscription.

## `ChargeOutcome` variants and keeper actions

[`contracts/subscription/src/types.rs`](../../contracts/subscription/src/types.rs#L87-L110)

| Variant | Meaning | Keeper action |
|---|---|---|
| `Charged` | A due charge succeeded; `last_charged` updated. | None required. Log it for your own accounting/metrics; a `SubRenewed` event was also emitted if you're indexing events separately. |
| `NotDueYet` | The interval (or trial period, for a first charge) hasn't elapsed. | None — this is the expected, no-op outcome for a subscription you included in the batch speculatively or ahead of schedule. Don't retry sooner; recompute its next-due timestamp from `last_charged + plan.interval` (or `created_at + plan.trial_period` if never charged) instead. |
| `EnteredGrace` | The charge failed for insufficient allowance and the subscription moved into (or remains in) `PastDue`, because `plan.grace_period > 0`. | Flag the subscription for customer-facing follow-up (dunning email, in-app prompt to re-authorize — see [Subscription billing authorization and allowances](../concepts/subscription-billing-authorization.md)). Keep including it in future sweeps — it needs to be re-attempted for recovery or eventual termination to happen at all. No event is emitted for this transition (see [Grace periods § Events](../concepts/subscription-grace-periods.md#events)), so this return value, or a direct `get_subscription` poll, is the only on-chain-adjacent signal you'll get. |
| `Recovered` | A `PastDue` subscription was successfully charged and returned to `Active`. | Clear any dunning flag you set for `EnteredGrace`. A `SubRenewed` event was emitted. |
| `Terminated` | The grace window expired during this call without a successful charge; subscription is now permanently `Terminated`. | Revoke access immediately (off-chain) and stop including this `sub_id` in future sweeps — it can never be charged or recovered again. A `SubExpired` event was emitted. |
| `Skipped` | The subscription was already `Cancelled` or `Terminated` when this batch touched it — only returned by `process_billing_cycle` (`process_charge` panics instead). | Drop this `sub_id` from your tracked set; it's dead weight in every future sweep. |

## Scheduling strategy

### Discovering due subscriptions

The `subscription` contract emits **no event when a subscription is created** — neither `subscribe` nor `subscribe_with_token` publishes anything (confirmed against [`contracts/subscription/src/lib.rs`](../../contracts/subscription/src/lib.rs#L224-L298): both functions write storage and return the new `sub_id`, with no `.publish()` call). This means you cannot build your initial tracked set purely by indexing events the way [Event indexing guide](../guides/indexing-events.md) describes for the Shade hub. Two viable approaches, often used together:

1. **Capture `sub_id` at creation time.** Whatever application flow calls `subscribe`/`subscribe_with_token` already receives the new subscription ID as the call's return value — persist it into your own keeper database the moment it's created, rather than trying to discover it later.
2. **Enumerate from `get_subscription_count()`.** `get_subscription_count` returns the total number of subscriptions ever created, and IDs are sequential starting at 1 — a keeper can periodically scan `1..=get_subscription_count()`, calling `get_subscription(id)` for any ID it doesn't already have tracked, to catch anything missed by approach 1. This is a reasonable one-time or periodic reconciliation pass, not something to run on every sweep given the read cost of scanning every ID.

Once a subscription is tracked, use `SubRenewed`/`SubExpired` events (see [Events reference § `subscription` contract](../reference/events.md#subscription-contract)) as a cheap incremental signal for "this one changed," falling back to `get_subscription` reads to disambiguate exactly what changed, since both the `Charged`→`Recovered` success path and `Cancelled`→... no — `SubExpired` in particular is emitted for both cancellation and grace-expiry termination, so the event alone can't tell you which happened; read `status` to tell them apart.

For each tracked, non-terminal subscription, compute its next due timestamp from `get_subscription` + `get_plan`:

```text
due_at = (last_charged == 0)
  ? created_at + trial_period
  : last_charged + plan.interval
```

A subscription is a candidate for the next sweep once `now >= due_at`. Include `PastDue` subscriptions in every sweep regardless of `due_at` — they're already overdue and need repeated attempts to resolve into `Recovered` or `Terminated`.

### Batch sizing against Soroban resource limits

`process_billing_cycle` takes an unbounded `Vec<u64>` — there is no `MAX_BATCH_SIZE` constant or check anywhere in [`contracts/subscription/src/lib.rs`](../../contracts/subscription/src/lib.rs). The real ceiling is Soroban's per-transaction resource metering (CPU instructions, memory, and ledger entries read/written in one invocation) — the same constraint [Payment flows § Walkthrough: `pay_invoices_batch`](../concepts/payments.md#walkthrough-pay_invoices_batch) describes for the hub's own batch payment function. Each entry in the batch performs at minimum a `Subscription` read, a `Plan` read, a token `allowance` read, and (if charging) a `transfer_from` plus a `Subscription` write and an event publish — so the safe batch size depends on your deployment's actual resource budget, not a number this doc can hand you. Start conservative (tens of subscriptions per call, not hundreds), measure the simulated resource usage the RPC reports back for a representative batch, and size up from there with margin — a batch that exceeds the budget fails the entire transaction with no partial progress, which is strictly worse than just running more, smaller batches.

### Run cadence

Run the sweep more frequently than your shortest plan's billing interval, not just once per interval — a keeper that only wakes up once per cycle has no slack for retrying a transient RPC failure or a transaction that failed to land before the next due date. A reasonable starting cadence is hourly for day-or-longer intervals, scaling down for any plan with a materially shorter interval. There's no contract-side reason a sweep can't run more often than needed — `NotDueYet` is a free, side-effect-free no-op (see [Idempotency](#idempotency)).

## Idempotency

Re-running a batch — including one that partially succeeded, or one you're not sure landed — is safe:

- **`NotDueYet`** is returned, not acted on, whenever `now` is still before the computed due timestamp. Calling `process_charge`/`process_billing_cycle` again before the interval elapses changes nothing and charges nothing twice.
- **`Skipped`** is returned for any already-`Cancelled`/`Terminated` subscription in a batch — re-including a dead ID in every future sweep (until you prune it per [Discovering due subscriptions](#discovering-due-subscriptions)) costs a read, not a double charge.
- **A charge updates `last_charged` to the ledger's current timestamp before anything else can observe it mid-transaction**, and each `charge`/`process_charge`/`process_billing_cycle` invocation is its own atomic Soroban transaction — there's no window where two concurrent keeper runs can both see "due" and both successfully pull funds for the same cycle: whichever lands first advances `last_charged`, and the second attempt (whether itself or a retry) re-evaluates against the new `last_charged` and gets `NotDueYet`.

Together, this means a keeper can safely retry an uncertain transaction (dropped connection, ambiguous confirmation) by simply re-submitting the same call — there is no dedicated idempotency-key mechanism because none is needed; the interval check in `step_billing_cycle` already provides it.

## Monitoring and alerting

- **Failed transactions.** Any `charge`/`process_charge`/`process_billing_cycle` call that fails at the transaction layer (expired footprint, insufficient fee, bad sequence number — see [Errors reference § Transaction-layer failures](../reference/errors.md)) produced no on-chain effect and no event; alert on these directly from your submission client's response, since there is nothing on-chain to reconcile against afterward.
- **Rising `EnteredGrace` counts.** Track how many subscriptions return `EnteredGrace` per sweep, and especially track subscriptions that have been `PastDue` across multiple consecutive sweeps without recovering — a sustained rise can indicate a token-side problem (customers' balances genuinely drained) or, as important, a keeper-side problem (your own retry logic failing to actually resubmit authorizations, or a bug in how you compute `due_at`). Since no event marks grace entry, this metric must come from your own logged `ChargeOutcome` values, not from an event stream.
- **Keeper account balance for fees.** The account submitting `charge`/`process_charge`/`process_billing_cycle` transactions pays the Soroban transaction fee for every call regardless of outcome (`Charged`, `NotDueYet`, or anything else) — a keeper that runs out of its own native-token balance stops being able to submit transactions at all, silently halting every subscription's billing simultaneously. Alert well before this account's balance is exhausted, not after.
- **Termination rate as a lagging indicator.** `Terminated` outcomes are the cost of unresolved grace — if your `EnteredGrace` alerting above is working, terminations should be rare and expected (customers who genuinely churned), not a surprise. A termination rate that doesn't track your grace-period alerting is a sign the monitoring pipeline itself has a gap.

## Reference keeper script outline

The outline below follows the generated TypeScript client pattern from [Generating and using contract bindings](../guides/contract-bindings.md). This is illustrative structure, not a drop-in implementation — there is no keeper implementation in this repository today; a real one would most naturally live under a new top-level directory (e.g. `keepers/subscription-billing/`) alongside its own `package.json`, independent of the Rust contract workspace.

```typescript
import { Client } from "../bindings/subscription/src/index.js";

const client = new Client({
  contractId: process.env.SUBSCRIPTION_CONTRACT_ID!,
  networkPassphrase: process.env.NETWORK_PASSPHRASE!,
  rpcUrl: process.env.RPC_URL!,
});

interface TrackedSubscription {
  subId: bigint;
  planId: bigint;
  status: "Active" | "PastDue" | "Cancelled" | "Terminated";
  dueAt: bigint; // computed off-chain from last_charged/created_at + plan interval/trial
}

async function discoverNewSubscriptions(db: KeeperDb) {
  // Reconciliation pass — see "Discovering due subscriptions" above.
  // Run periodically, not every sweep.
  const { result: count } = await client.get_subscription_count();
  for (let id = 1n; id <= count; id++) {
    if (await db.isTracked(id)) continue;
    const { result: sub } = await client.get_subscription({ sub_id: id });
    await db.track(sub);
  }
}

async function sweep(db: KeeperDb) {
  const candidates = await db.dueOrPastDue(); // your own query, per "Scheduling strategy"
  const BATCH_SIZE = 50; // start conservative; tune against simulated resource usage

  for (let i = 0; i < candidates.length; i += BATCH_SIZE) {
    const batch = candidates.slice(i, i + BATCH_SIZE);
    const subIds = batch.map((s) => s.subId);

    const tx = await client.process_billing_cycle({ sub_ids: subIds });
    const { result: outcomes } = await tx.signAndSend({ signTransaction: keeperSigner });

    outcomes.forEach((outcome, idx) => {
      const sub = batch[idx];
      switch (outcome.tag) {
        case "Charged":
        case "Recovered":
          db.markResolved(sub.subId);
          break;
        case "NotDueYet":
          break; // no-op, re-evaluate next sweep
        case "EnteredGrace":
          db.flagForDunning(sub.subId); // see monitoring: track consecutive-grace count
          break;
        case "Terminated":
          db.revokeAccess(sub.subId); // off-chain access control
          db.untrack(sub.subId);
          break;
        case "Skipped":
          db.untrack(sub.subId); // already Cancelled/Terminated
          break;
      }
    });
  }
}

// Run on an interval shorter than your shortest plan's billing interval.
setInterval(() => sweep(db).catch(alertOnFailure), SWEEP_INTERVAL_MS);
```

`keeperSigner` should be a `signTransaction` callback backed by a secret held in a secret manager, never inlined — see the [contract bindings guide's signing note](../guides/contract-bindings.md#using-the-generated-client-in-an-application). The keeper account itself needs no special authorization on the subscription contract — `process_billing_cycle` and `process_charge` require no caller auth at all (only the customer's own token allowance gates the actual fund movement), so any funded account can run this sweep.

## Related pages

- [Subscription contract reference](../contracts/subscription.md) — full type, function, lifecycle, and error listing.
- [Subscription grace periods and past-due handling](../concepts/subscription-grace-periods.md) — the state machine this keeper drives, including which transitions emit events.
- [Subscription billing authorization and allowances](../concepts/subscription-billing-authorization.md) — how customers grant and restore the allowance a keeper's charge attempts depend on.
- [Payment flows: full, partial, and batch § Walkthrough: `pay_invoices_batch`](../concepts/payments.md#walkthrough-pay_invoices_batch) — the same Soroban resource-metering ceiling discussed here, for the hub's own batch payment function.
- [Events reference § `subscription` contract](../reference/events.md#subscription-contract) — `SubRenewed`/`SubExpired` payloads.
- [Generating and using contract bindings](../guides/contract-bindings.md) — the TypeScript client pattern the keeper outline above builds on.
