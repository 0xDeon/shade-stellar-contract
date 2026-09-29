# Event indexing guide

How to build a reliable off-chain indexer over Shade's on-chain events: the data source, a recommended schema, the ingestion loop, gap detection, backfills, and reconciliation against on-chain state.

Shade does not ship an indexer in this repository. This guide documents the RPC mechanism the contract's events are observable through and a recommended design for consuming them — there is no existing indexer implementation here to link to or reuse.

## 1. Event data source

Soroban events are read through the RPC method `getEvents` on a `soroban-rpc`-compatible endpoint (the same RPC surface the Stellar/Soroban CLI and SDKs use). This is a Stellar-platform mechanism, not something Shade defines — see [Stellar docs: Contract Interactions](https://developers.stellar.org/docs/build/guides/conventions) and the [Stellar RPC API reference](https://developers.stellar.org/docs/data/rpc/api-reference/methods/getEvents) for the authoritative method specification. This guide documents how to use it against Shade specifically, not the RPC protocol itself.

### Request shape

`getEvents` takes, at minimum:

- **`startLedger`** — the ledger sequence to begin from (exclusive of history before it).
- **`filters`** — an array of filters, each of which can constrain by `type` (`contract`, `system`, or `diagnostic`), `contractIds` (restrict to the `shade` contract's address), and `topics` (match on the event's topic vector — the first topic is the event name Symbol, e.g. the snake_case form of `MerchantWebhookSetEvent`).
- **`pagination`** — a `cursor` and `limit` for paging through results.

A minimal filter for this workspace: `contractIds: [$SHADE_ID]` with no topic filter, if you want every Shade event, or a topic filter naming a specific event if you only care about one type.

### Response shape and pagination

The response returns an array of event objects, each carrying (per the Stellar RPC spec): the contract ID, the ledger sequence, the ledger close time, the transaction hash, the event's topics, its value (XDR-encoded), and a paging `id`/cursor you pass back in the next request's `pagination.cursor` to continue. Exactly which field names your specific RPC provider/SDK version surfaces for the per-event ordering position (sometimes exposed as part of the cursor, sometimes as an explicit index field) varies by `soroban-rpc` version — confirm against the response your specific provider returns before hard-coding a field name into your ingestion code, rather than assuming this guide's description matches byte-for-byte.

### Why the RPC's retained history is not an archival database

`getEvents` only serves events within the RPC node's configured retention window (commonly measured in days, and configurable per RPC operator) — see [Stellar docs: State archival](https://developers.stellar.org/docs/learn/encyclopedia/storage/state-archival) for the general retention model this follows. An indexer that only ever calls `getEvents` with a recent `startLedger` will silently lose access to older events once they fall out of the node's retention window. This is why [Backfills](#8-backfills) below distinguishes a live ingestion cursor from a separate archival backfill process: the RPC node is a rolling window onto recent history, not a permanent log, and your indexer's own database is the thing that must become the permanent record.

## 2. Event catalog

Every event `#[contractevent]` struct relevant to merchants, invoices, payments, refunds, subscriptions, and tickets, from [`contracts/shade/src/events.rs`](../../contracts/shade/src/events.rs). All are emitted by the `shade` contract; there is currently no separate on-chain event emission for merchant, invoice, or payment activity from the standalone `account`, `escrow`, `subscription`, or `ticketing` crates when they are used independently of the hub.

| Event | Business entity | Payload fields | Ordering considerations |
|---|---|---|---|
| `MerchantRegisteredEvent` | Merchant | `merchant`, `merchant_id`, `timestamp` | One per merchant, at most once (registration is a single-use action per address). |
| `MerchantAccountDeployedEvent` | Merchant | `merchant`, `contract`, `timestamp` | Emitted in the same transaction as `MerchantRegisteredEvent`, immediately after. |
| `MerchantStatusChangedEvent` | Merchant | `merchant_id`, `active`, `timestamp` | Can occur any number of times per merchant; only the latest value is authoritative for current state. |
| `MerchantVerifiedEvent` | Merchant | `merchant_id`, `status`, `timestamp` | Same pattern as status — latest value wins. |
| `MerchantWebhookSetEvent` | Merchant | `merchant`, `merchant_id`, `webhook`, `timestamp` | Latest value wins; historical values are still visible in the event log (see the privacy note in [Webhooks](webhooks.md#on-chain-webhook-configuration)). |
| `MerchantKeySetEvent` | Merchant | `merchant`, `key`, `timestamp` | Latest value wins; rotating a key invalidates prior signed-invoice payloads (see [Signed invoices](../security/signatures.md)). |
| `MerchantTokensSetEvent` | Merchant | `merchant`, `tokens`, `timestamp` | Full replacement of the merchant's accepted-token list each time — not a diff. |
| `MerchantTokenRemovedEvent` | Merchant | `merchant`, `token`, `timestamp` | Emitted by `remove_merchant_accepted_token`, a single-token removal distinct from a full `set_merchant_accepted_tokens` replace. |
| `InvoiceCreatedEvent` | Invoice | `invoice_id`, `merchant`, `amount`, `token` | No `timestamp` field — use the RPC response's ledger close time if you need wall-clock time for this event specifically. |
| `InvoicePaidEvent` | Payment | `invoice_id`, `merchant_id`, `merchant_account`, `payer`, `amount`, `fee`, `merchant_amount`, `token`, `timestamp` | Emitted by both `pay_invoice` (full payment) and `pay_invoice_partial` — see the [partial-payment note](webhooks.md#per-event-type-payload-mapping) in the webhooks guide. Also emitted once per invoice inside `pay_invoices_batch`. |
| `PlatformFeeRoutedEvent` | Payment | `route_kind`, `ref_id`, `merchant_id`, `merchant`, `merchant_account`, `platform_account`, `payer`, `gross_amount`, `platform_fee`, `merchant_amount`, `token`, `fee_bps_applied`, `timestamp` | Emitted alongside `InvoicePaidEvent`/`SubscriptionChargedEvent`; `route_kind` (a `PlatformFeeRouteKind` cast to `u32`) and `ref_id` tell you which domain (invoice vs. subscription vs. ticket) it corresponds to — use this event, not `InvoicePaidEvent` alone, if you need a single canonical fee-split record across all payment types. |
| `InvoiceRefundedEvent` | Refund | `invoice_id`, `merchant`, `amount`, `timestamp` | Full refund only; see `InvoicePartiallyRefundedEvent` for partial. |
| `InvoicePartiallyRefundedEvent` | Refund | `invoice_id`, `merchant`, `amount`, `total_amount_refunded`, `timestamp` | `total_amount_refunded` is cumulative across all partial refunds on this invoice, not just this event's `amount`. |
| `InvoiceCancelledEvent` | Invoice | `invoice_id`, `merchant`, `timestamp` | This is the event `void_invoice` emits — there is no separately named "voided" event. |
| `InvoiceAmendedEvent` | Invoice | `invoice_id`, `merchant`, `old_amount`, `new_amount`, `timestamp` | Only fires for unfinalized (draft) invoices via `amend_invoice`. |
| `SubscriptionPlanCreatedEvent` | Subscription | `plan_id`, `merchant`, `token`, `amount`, `interval`, `timestamp` | One per plan. |
| `SubscribedEvent` | Subscription | `subscription_id`, `plan_id`, `customer`, `timestamp` | One per subscription creation. |
| `SubscriptionChargedEvent` | Payment | `subscription_id`, `plan_id`, `customer`, `merchant`, `amount`, `fee`, `token`, `timestamp` | Recurs once per successful `charge_subscription` call; `PlatformFeeRoutedEvent` also fires alongside it. |
| `SubscriptionCancelledEvent` | Subscription | `subscription_id`, `caller`, `timestamp` | `caller` may be either the customer or the merchant — both are permitted to cancel. |
| `PlanDeactivatedEvent` | Subscription | `plan_id`, `merchant`, `timestamp` | Does not cancel existing subscriptions on the plan; only blocks new enrollment. |
| `EventCreatedEvent` | Ticket (event) | `event_id`, `merchant`, `merchant_id`, `name`, `ticket_price`, `token`, `capacity`, `event_date`, `royalty_bps`, `timestamp` | One per ticketed event. |
| `TicketPurchasedEvent` | Ticket | `ticket_id`, `event_id`, `merchant_id`, `buyer`, `amount`, `fee`, `merchant_amount`, `token`, `timestamp` | One per ticket, including each ticket within a `purchase_tickets_bulk` call. |
| `TicketResoldEvent` | Ticket | `ticket_id`, `event_id`, `merchant_id`, `seller`, `buyer`, `resale_price`, `royalty`, `seller_proceeds`, `token`, `timestamp` | Secondary-market resale; does not affect original `TicketPurchasedEvent` history. |
| `FiatInvoicePricedEvent` | Invoice | `invoice_id`, `token`, `resolved_amount`, `timestamp` | Only for fiat-denominated invoices, at the moment of oracle-price resolution. |

Configuration-adjacent events worth tracking even though they are not a business entity above: `TokenAddedEvent`/`TokenRemovedEvent` (global accepted-token whitelist changes — affects which tokens `is_token_accepted_for_merchant` will allow by default), and `ContractUpgradedEvent` (see [Upgradeability](../architecture/upgradeability.md#establishing-what-is-deployed) — track this if your indexer needs to know when the contract's behavior may have changed).

## 3. Relational schema

A recommended schema. Adapt field types to your database; `i128` amounts should map to a type with at least 128-bit precision (e.g. Postgres `numeric`), since a `bigint`/`int8` will silently truncate large values.

### merchants

| Column | Type | Source |
|---|---|---|
| `merchant_id` | bigint, primary key | `MerchantRegisteredEvent.merchant_id` |
| `address` | text | `MerchantRegisteredEvent.merchant` |
| `account_address` | text, nullable | `MerchantAccountDeployedEvent.contract`, or a periodic `get_merchant_account` read (event-derived at deploy time; reconcile if `set_merchant_account` is ever called) |
| `active` | boolean | Latest `MerchantStatusChangedEvent.active`, defaults `true` at registration |
| `verified` | boolean | Latest `MerchantVerifiedEvent.status`, defaults `false` |
| `webhook_url` | text, nullable | Latest `MerchantWebhookSetEvent.webhook` |
| `first_seen_ledger` | bigint | Ledger of `MerchantRegisteredEvent` |
| `last_updated_ledger` | bigint | Ledger of the most recent event touching this merchant |

`active`, `verified`, and `webhook_url` are **event-derived but reconciled** — see [Reconciliation](#11-reconciliation): confirm periodically against `is_merchant_active`, `is_merchant_verified`, and `get_merchant_webhook` reads, since these are simple latest-value-wins fields where a missed event would silently desync the index.

### invoices

| Column | Type | Source |
|---|---|---|
| `invoice_id` | bigint, primary key | `InvoiceCreatedEvent.invoice_id` |
| `merchant_id` | bigint | Resolved from `InvoiceCreatedEvent.merchant` (an `Address`) via the `merchants` table, or a `find_merchant_id` read if the merchant isn't indexed yet |
| `token` | text | `InvoiceCreatedEvent.token` |
| `amount` | numeric | `InvoiceCreatedEvent.amount` |
| `status` | text/enum | Derived: `Pending` at creation, `Paid`/`PartiallyPaid` on `InvoicePaidEvent`, `Cancelled` on `InvoiceCancelledEvent`, `Refunded`/`PartiallyRefunded` on refund events — mirror `InvoiceStatus` from [`contracts/shade/src/types.rs#L358-L366`](../../contracts/shade/src/types.rs#L358-L366) |
| `amount_paid` | numeric | Cumulative from `InvoicePaidEvent`s for this invoice |
| `amount_refunded` | numeric | Cumulative from `InvoicePartiallyRefundedEvent.total_amount_refunded`, or full `InvoiceRefundedEvent.amount` |
| `creation_ledger` | bigint | Ledger of `InvoiceCreatedEvent` |
| `paid_at_ledger` | bigint, nullable | Ledger of the qualifying `InvoicePaidEvent` |
| `last_event_id` | text | The idempotency key (see [Idempotency](#4-idempotency)) of the most recent event applied to this row |

`status` is **event-derived**; reconcile periodically against `get_invoice` (see [Reconciliation](#11-reconciliation)) since it's a state machine built entirely from your own event replay logic and a bug there desyncs silently.

### payments

| Column | Type | Source |
|---|---|---|
| `event_key` | text, primary key | Idempotency key — see [Idempotency](#4-idempotency) |
| `invoice_id` | bigint | `InvoicePaidEvent.invoice_id` |
| `payer` | text | `InvoicePaidEvent.payer` |
| `amount` | numeric | `InvoicePaidEvent.amount` |
| `fee` | numeric | `InvoicePaidEvent.fee` |
| `merchant_amount` | numeric | `InvoicePaidEvent.merchant_amount` |
| `token` | text | `InvoicePaidEvent.token` |
| `ledger` | bigint | Ledger the event was emitted in |
| `tx_hash` | text | Transaction hash from the RPC event response |

One row per `InvoicePaidEvent`, including each partial payment — this table is the append-only ledger of individual payments; `invoices.amount_paid` is the derived running total.

### refunds

| Column | Type | Source |
|---|---|---|
| `event_key` | text, primary key | Idempotency key |
| `invoice_id` | bigint | `InvoiceRefundedEvent.invoice_id` / `InvoicePartiallyRefundedEvent.invoice_id` |
| `amount` | numeric | `.amount` from either event |
| `is_partial` | boolean | `true` for `InvoicePartiallyRefundedEvent`, `false` for `InvoiceRefundedEvent` |
| `ledger` | bigint | Ledger the event was emitted in |
| `tx_hash` | text | Transaction hash |

### subscriptions

| Column | Type | Source |
|---|---|---|
| `subscription_id` | bigint, primary key | `SubscribedEvent.subscription_id` |
| `plan_id` | bigint | `SubscribedEvent.plan_id` |
| `customer` | text | `SubscribedEvent.customer` |
| `merchant_id` | bigint | Resolved from the plan's merchant (via `SubscriptionPlanCreatedEvent.merchant` or a `get_subscription_plan` read) |
| `status` | text/enum | Derived: active at `SubscribedEvent`, cancelled at `SubscriptionCancelledEvent` — mirror `SubscriptionStatus` |
| `last_charge_ledger` | bigint, nullable | Ledger of the most recent `SubscriptionChargedEvent` for this subscription |
| `last_charge_amount` | numeric, nullable | `SubscriptionChargedEvent.amount` |

### tickets

| Column | Type | Source |
|---|---|---|
| `ticket_id` | bigint, primary key | `TicketPurchasedEvent.ticket_id` |
| `event_id` | bigint | `TicketPurchasedEvent.event_id` |
| `merchant_id` | bigint | `TicketPurchasedEvent.merchant_id` |
| `owner` | text | Latest holder — `TicketPurchasedEvent.buyer` at mint, updated to `TicketResoldEvent.buyer` on resale |
| `amount` | numeric | `TicketPurchasedEvent.amount` (original purchase) |
| `token` | text | `TicketPurchasedEvent.token` |
| `purchase_ledger` | bigint | Ledger of `TicketPurchasedEvent` |

## 4. Idempotency

**Primary idempotency key:** `{contract_id}:{ledger}:{tx_hash}:{event_index_within_tx}` (or the ledger + event-index form described in [Webhooks: Idempotency](webhooks.md#idempotency) if your RPC provider exposes a clean per-ledger event index directly). Verify exactly what fields your specific `soroban-rpc` provider/SDK version returns per event before finalizing which of these you compose the key from — see [the data source section above](#event-data-source).

**Why event payload values alone are insufficient:** two distinct on-chain events can carry identical payload values — two `InvoiceCreatedEvent`s for two different merchants could coincidentally have the same `amount` and `token`; a subscription charged twice at the same recurring `amount` produces two `SubscriptionChargedEvent`s with identical fields except for ledger position. Only the ledger/transaction/event-index identity is guaranteed unique per emission.

**Enforce with a database unique constraint**, not just an application-level check:

```sql
CREATE UNIQUE INDEX idx_payments_event_key ON payments (event_key);
```

A unique constraint catches a duplicate insert even if two ingestion workers race, or if a crash-and-replay (see [Cursor persistence](#5-cursor-persistence)) re-processes a range that was already partially committed. An application-level "check if it exists, then insert" has a race window a constraint does not.

## 5. Cursor persistence

Persist, atomically with the business-state write for the same batch:

- **Last processed ledger** — the highest ledger sequence fully processed.
- **Last processed event identity** — the idempotency key of the last event applied, for resuming mid-ledger if a ledger contains many events.

**Atomic persistence:** write the cursor update in the **same database transaction** as the business-state upserts for that batch (see [Ingestion loop](#6-ingestion-loop) step 6–7). If the cursor commits without the business state, a crash-and-restart re-reads events already applied — harmless, because of the idempotency constraint from step 4. If the business state commits without the cursor, a restart skips those events entirely on the next run — this is the failure mode to avoid.

**Restart behavior:** on startup, read the persisted cursor and resume `getEvents` from `startLedger = last_processed_ledger` (re-processing that ledger is safe and expected, not an off-by-one bug, given idempotent upserts).

**The invariant to preserve:**

> A crash must cause replay/reprocessing at worst, never silent event loss.

This is why cursor and business-state commit together: any ordering that allows the cursor to advance past events that were never actually applied violates this invariant.

## 6. Ingestion loop

```text
1. load persisted cursor (last_processed_ledger, last_processed_event_key)
2. query getEvents(startLedger = last_processed_ledger, filters = [...], cursor = ...)
3. for each returned event:
   a. validate/normalize (decode XDR payload, map topic to event type)
   b. skip if event_key <= last_processed_event_key (defense in depth alongside the unique constraint)
4. begin database transaction
5. for each validated event:
   a. idempotently upsert business state (merchants/invoices/payments/... per the schema above)
      — use INSERT ... ON CONFLICT (event_key) DO NOTHING for append-only tables (payments, refunds)
      — use INSERT ... ON CONFLICT (id) DO UPDATE for latest-value-wins tables (merchants.active, invoices.status)
6. persist updated cursor in the same transaction
7. commit
8. request next page using the RPC's returned pagination cursor
9. repeat from step 2
```

The database transaction boundary in steps 4–7 is what makes step 5's per-event idempotency and step 6's cursor update atomic with each other — this is the mechanism, not just a suggestion, behind the crash-safety invariant in [Cursor persistence](#5-cursor-persistence).

## 7. Gap detection

Signals that indicate a gap rather than a clean, complete stream:

- **Missing ledger ranges:** if your indexer's `last_processed_ledger` and the RPC's current ledger differ by more than expected given your polling interval, you may have missed a polling cycle (downtime, crash before restart, deployment gap).
- **RPC pagination anomalies:** a `getEvents` response whose cursor does not advance, or that returns fewer events than the requested `limit` while claiming more exist, can indicate a provider-side inconsistency — treat this defensively rather than assuming the stream is exhausted.
- **Cursor corruption:** if the persisted cursor is unreadable, corrupted, or inconsistent with the business-state tables (e.g. `last_processed_ledger` is lower than the highest ledger already present in `payments`), stop ingesting and investigate rather than guessing a resume point.
- **Service downtime:** any period the ingestion process was not running is a gap by definition — track process uptime/heartbeats separately from the cursor so you can bound how large a gap might be.
- **Retention windows:** if downtime (or a delayed first deployment) exceeds the RPC node's retention window (see [Event data source](#event-data-source)), the live cursor **cannot** resume from where it left off — that range is permanently outside `getEvents`'s reach and requires the backfill path below instead.
- **Incomplete historical coverage:** if the indexer's live cursor started at "now" rather than at the contract's actual deployment/initialization ledger, everything before that start is a permanent, known gap unless backfilled.

When any of the above is detected: **stop normal live ingestion for the affected range** rather than continuing forward and leaving a silent hole, and trigger the backfill procedure in the next section for that specific range.

## 8. Backfills

- **Initial historical backfill:** on first deployment, backfill from the contract's `initialize` ledger (or however far back your archival source reaches) forward to the point the live cursor begins, using the same `getEvents` mechanism against an archive-capable RPC provider if the range exceeds the live RPC node's retention window.
- **Periodic archival backfill:** even with a healthy live cursor, periodically re-verify older ranges are still consistent (see [Reconciliation](#11-reconciliation)) — an archival backfill is a targeted re-run of the ingestion loop over a historical range, not a special code path.
- **Recovery after downtime:** run a backfill over exactly the gap range identified in [Gap detection](#7-gap-detection), then resume the live cursor from where it left off.
- **Backfill beyond RPC retention:** requires a data source other than the live RPC node's `getEvents` — either a different archive-tier RPC provider, or, if no such provider covers the range, accept that range as permanently unindexable at the event level and rely on periodic on-chain state reads (see [Reconciliation](#11-reconciliation)) as the closest available substitute for current-state accuracy.
- **Deduplication during backfill:** rely on the same idempotency mechanism as live ingestion ([Idempotency](#4-idempotency)) — a backfill range that overlaps already-indexed ledgers must be safe to re-run, which the unique-constraint approach guarantees.
- **Reconciliation after backfill:** run the reconciliation procedure ([below](#11-reconciliation)) over the backfilled range once complete, to confirm the backfilled event-derived state matches current on-chain reads.

**Why the live cursor and historical backfill must be coordinated:** if a backfill job and the live ingestion loop process overlapping ledger ranges without a shared idempotency mechanism, you double-count payments and inflate volume/analytics. If they're coordinated so poorly that neither covers a range, you get a silent gap instead. The unique-constraint approach in [Idempotency](#4-idempotency) is what lets both processes run against overlapping ranges safely — treat any backfill job as "just another consumer" of the same idempotent-upsert logic, not a separate code path.

## 9. Failed transactions

**Distinguish, using the RPC/transaction-result data, not the presence of an event alone:**

- **Submitted transaction** — a transaction was sent to the network. This alone tells you nothing about outcome.
- **Successful transaction** — the transaction's result indicates success. Soroban events are only emitted for the operations that actually executed as part of a successful contract invocation.
- **Failed transaction** — the transaction failed (host trap, insufficient fee, expired footprint, etc.). A failed transaction's attempted contract calls do **not** emit the events the contract logic would have emitted on success — a `pay_invoice` call that panics with `InsufficientBalance` never reaches the `events::publish_invoice_paid_event` call in [`contracts/shade/src/components/invoice.rs`](../../contracts/shade/src/components/invoice.rs), so no `InvoicePaidEvent` exists for it to index.
- **Do not infer a successful payment merely because a transaction contained a payment invocation.** If you are inspecting transactions directly (rather than only consuming `getEvents`, which by construction only returns events from successful executions), always check the transaction result's success/failure status before treating any contained operation as having taken effect.

In practice, an indexer built purely on `getEvents` (as this guide recommends) does not need special-case failed-transaction handling for its own correctness, because `getEvents` only ever returns events that were actually emitted by successful executions — the risk above applies specifically if you additionally ingest raw transactions or ledger close metadata directly and try to infer business events from operation lists rather than from the emitted event log.

## 10. Reorg / ledger finality

**Classic blockchain "reorg" handling does not apply to Stellar/Soroban the way it does on Ethereum-style probabilistic-finality chains.** Stellar uses the Stellar Consensus Protocol, under which a ledger that has closed and been confirmed by validators is final — there is no probabilistic-finality window during which a later, competing chain can supersede an already-closed ledger the way a PoW reorg can. Do not build reorg-rollback logic modeled on Ethereum's "wait N confirmations, then be prepared to revert" pattern; it does not correspond to a real failure mode here.

**What can actually happen, and what to do about it:**

- **An RPC/provider exposing an inconsistent or transient view** — for instance, a load-balanced RPC cluster where one node is slightly behind another, or a node briefly returning stale data during its own catch-up — is a **data-consistency problem with your RPC provider**, not a protocol-level reorg. Handle it by: querying a consistent provider (a single node, or a provider that guarantees consistent reads across its cluster), and treating a `getEvents` response with an ledger that regresses versus a previous response as a signal to re-fetch rather than to trust the data as final.
- **A transaction you submitted appearing to fail, then later appearing to have succeeded (or vice versa)** during transient RPC inconsistency should be resolved by querying transaction status directly against a specific, trusted RPC node rather than inferring finality from event presence alone during the inconsistency window.
- The correct model to hold onto: **once a ledger has closed on Stellar, its contents do not change.** Treat any apparent contradiction after the fact as evidence of an RPC-layer inconsistency to investigate and route around, not as a chain-level event requiring rollback logic in your indexer.

## 11. Reconciliation

A periodic procedure comparing your indexed, event-derived state against authoritative on-chain reads, using confirmed-to-exist read functions: `get_invoice` and `get_merchant_analytics` (see [ShadeTrait function reference](../reference/shade-interface.md#get_invoice) and [`#get_merchant_analytics`](../reference/shade-interface.md#get_merchant_analytics)).

1. **Select a reconciliation window/sample** — e.g. all invoices created in the last 24 hours, plus a random sample of older ones, rather than every record every run (cost-prohibitive at scale).
2. **Fetch indexed state** — read the sampled rows from your `invoices` and merchant-analytics-adjacent tables.
3. **Query authoritative on-chain state** — call `get_invoice(invoice_id)` for each sampled invoice, and `get_merchant_analytics(merchant, token)` for each sampled merchant/token pair.
4. **Compare** — `status`, `amount_paid`, `amount_refunded` on invoices; `total_volume`, `total_fees`, `transaction_count` on merchant analytics.
5. **Classify drift:**
   - **Missing entirely** in the index but present on-chain — a gap (see [Gap detection](#7-gap-detection)); trigger a targeted backfill.
   - **Present but stale** (index shows `Pending`, chain shows `Paid`) — an event was likely missed or mis-processed; re-run ingestion for that record's ledger range.
   - **Present and matching** — no action.
6. **Repair through reindex/backfill/read refresh** — for drifted records, either re-run the ingestion loop over the specific ledger range that should have produced the missing event, or, for a purely current-state field with no practical event-replay path, refresh directly from the on-chain read and record that the value was read-repaired rather than event-derived for that instance.
7. **Record the reconciliation result** — window checked, records compared, drift found, repairs applied — so reconciliation health itself is auditable over time.

**Distinguish event-derived historical facts from current state reads** throughout: `payments` and `refunds` rows are historical facts your index reconstructs from the event log and can only be verified by re-examining that same log (there is no `get_payment_history` on-chain equivalent to re-derive them from). `invoices.status` and `merchant_analytics`-adjacent fields are **current state**, independently re-readable at any time via `get_invoice`/`get_merchant_analytics` — reconciliation is strongest for these current-state fields specifically, because they have an authoritative source you can re-query directly, unlike the historical event facts.

## 12. Worked minimal indexer example

A minimal example indexing just `InvoicePaidEvent`, in pseudocode following the loop in [section 6](#6-ingestion-loop). No live credentials or production endpoints — `$RPC_URL` and `$SHADE_ID` are placeholders for your own local/testnet values.

```python
import requests

RPC_URL = "http://localhost:8000/soroban/rpc"  # local standalone node; substitute your testnet RPC
SHADE_ID = "CA..."  # placeholder — your deployed shade contract id

def load_cursor(db):
    row = db.query_one("SELECT last_ledger, last_event_key FROM ingestion_cursor WHERE id = 1")
    return row or {"last_ledger": 0, "last_event_key": None}

def fetch_events(start_ledger, page_cursor=None):
    params = {
        "startLedger": start_ledger,
        "filters": [{
            "type": "contract",
            "contractIds": [SHADE_ID],
            "topics": [["invoice_paid_event"]],  # snake_case event-name topic
        }],
        "pagination": {"cursor": page_cursor, "limit": 100} if page_cursor else {"limit": 100},
    }
    resp = requests.post(RPC_URL, json={"jsonrpc": "2.0", "id": 1, "method": "getEvents", "params": params})
    resp.raise_for_status()
    return resp.json()["result"]

def event_key(evt):
    # Confirm your provider's actual per-event ordering field before relying on this in production —
    # see "Idempotency" above. This example assumes the response exposes a paging id usable as-is.
    return f"{SHADE_ID}:{evt['ledger']}:{evt['txHash']}:{evt['id']}"

def decode_invoice_paid(evt):
    # XDR-decode evt["value"] into the InvoicePaidEvent struct fields.
    # Left as a placeholder — the exact decode call depends on your SDK's XDR bindings.
    return decode_contract_event_xdr(evt["value"])

def ingest_batch(db, events):
    with db.transaction():
        max_ledger = None
        last_key = None
        for evt in events:
            key = event_key(evt)
            data = decode_invoice_paid(evt)
            db.execute("""
                INSERT INTO payments (event_key, invoice_id, payer, amount, fee, merchant_amount, token, ledger, tx_hash)
                VALUES (%(key)s, %(invoice_id)s, %(payer)s, %(amount)s, %(fee)s, %(merchant_amount)s, %(token)s, %(ledger)s, %(tx_hash)s)
                ON CONFLICT (event_key) DO NOTHING
            """, {"key": key, "ledger": evt["ledger"], "tx_hash": evt["txHash"], **data})
            max_ledger, last_key = evt["ledger"], key
        if max_ledger is not None:
            db.execute("""
                UPDATE ingestion_cursor SET last_ledger = %s, last_event_key = %s WHERE id = 1
            """, (max_ledger, last_key))

def run_once(db):
    cursor = load_cursor(db)
    result = fetch_events(start_ledger=cursor["last_ledger"])
    events = result.get("events", [])
    if events:
        ingest_batch(db, events)
    return result.get("cursor")  # pass to the next fetch_events call for pagination
```

This example deliberately omits: retry/backoff on RPC failure, backfill orchestration, and reconciliation — each is covered in its own section above and should be a separate, testable piece of the real implementation rather than inlined into the core loop.

## Related pages

- [Merchant webhooks and off-chain notifications](webhooks.md) — the notification layer built on top of this same event stream.
- [Errors reference](../reference/errors.md) — how to distinguish a contract error from a failed transaction when building reconciliation tooling.
- [Merchant Onboarding Guide](merchant-onboarding.md) — the events emitted during the flow this indexer would track.
- [Upgradeability](../architecture/upgradeability.md#establishing-what-is-deployed) — tracking `ContractUpgradedEvent` if your indexer needs to know when contract behavior changed.

← [Back to guides](README.md)
