# Refunds and voids

How `shade` reverses or amends an [invoice](../glossary.md#invoice) after creation: full refunds, partial refunds, voids, buyer-initiated expiry claims, and amendments. All behavior on this page is traced from [`contracts/shade/src/components/invoice.rs`](../../contracts/shade/src/components/invoice.rs) and cross-checked against [`contracts/shade/src/events.rs`](../../contracts/shade/src/events.rs) and [`contracts/shade/src/errors.rs`](../../contracts/shade/src/errors.rs).

## Decision table

| Situation | Operation | Authorized caller | Preconditions | Resulting status | Funds move? |
|---|---|---|---|---|---|
| Merchant wants to reverse a fully paid invoice, within the refund window | `refund_invoice` | `merchant` (must own the invoice) | Status `Paid`; `payer` set; `date_paid` set and `now - date_paid <= 604_800` (7 days); merchant account balance ≥ amount owed | `Refunded` | Merchant account → payer, full `amount - amount_refunded`. |
| Merchant wants to reverse part of a paid invoice | `refund_invoice_partial` | `merchant` | Status `Paid` or `PartiallyRefunded`; `date_paid` set and (if set) within the 7-day window; `amount > 0`; `amount_refunded + amount <= invoice.amount` | `Refunded` if the running total now equals `invoice.amount`, else `PartiallyRefunded` | Merchant account → payer, exactly `amount`. |
| Merchant wants to reverse a paid invoice again, after a prior partial refund | `refund_invoice_partial` | `merchant` | Same as above — `PartiallyRefunded` is an allowed starting status, so this can be called repeatedly | `Refunded` once the cumulative total reaches `invoice.amount`, else stays `PartiallyRefunded` | Merchant account → payer, exactly the new `amount`. |
| Merchant wants to cancel an invoice nobody has paid yet | `void_invoice` | `merchant` | Status `Pending` (only) | `Cancelled` | None. |
| Merchant wants to change the amount or description before anyone pays | `amend_invoice` | `merchant` | Status `Pending` (only) | Unchanged (`Pending`) | None. |
| Invoice expired while paid or partially paid, and the merchant never refunded it | `claim_refund` | `buyer` (the recorded `payer`) | `expires_at` is set and has passed; status is `Paid` or `PartiallyPaid` (not already `Refunded`); `amount_paid - amount_refunded > 0`; merchant account balance sufficient | `Refunded` | Merchant account → buyer, `amount_paid - amount_refunded`. |

Every row is a distinct function; there is no single "refund" entry point that branches internally. `merchant_address.require_auth()` (or `buyer.require_auth()` for `claim_refund`) is the first line of every one of these functions.

## Full refunds — `refund_invoice`

Verified at [`invoice.rs:443-514`](../../contracts/shade/src/components/invoice.rs#L443-L514) (`check_invoice_refund_eligibility` + `refund_invoice`).

- **Caller:** the invoice's owning merchant, verified by comparing `merchant::get_merchant_id(merchant_address)` against `invoice.merchant_id`; panics `NotAuthorized` on mismatch.
- **Allowed starting state:** `InvoiceStatus::Paid` only — `check_invoice_refund_eligibility` panics `InvoiceNotPaid` otherwise.
- **Refund window:** `elapsed = now - date_paid`; panics `RefundPeriodExpired` if `elapsed > MAX_REFUND_DURATION` (`604_800` seconds / 7 days, `invoice.rs:23`). If `date_paid` is unset (should not happen once `Paid`), it panics `InvoiceNotPaid`.
- **Amount:** `amount_to_refund = invoice.amount - invoice.amount_refunded`. For a `Paid` invoice this is always the invoice's face amount minus anything already refunded — since `Paid` is only reachable from a `refund_invoice_partial` cycle by resolving to `Refunded` (a terminal state this function can't re-enter), in practice this is the full `invoice.amount` the first time `refund_invoice` runs on a never-refunded `Paid` invoice.
- **Fee treatment:** the refund amount is computed purely from `invoice.amount`/`amount_refunded`. It does **not** re-read or subtract the platform fee taken at payment time — `refund_invoice` pays `amount_to_refund` entirely out of the merchant's own account. The platform fee collected at `pay_invoice` time (paid separately, straight to the platform account at payment time — see [`invoice.rs:736-738`](../../contracts/shade/src/components/invoice.rs#L736-L738)) is **not** returned to the merchant or clawed back from the platform account by `refund_invoice`. In other words: the platform fee is absorbed by the merchant, who must refund the buyer's full paid amount from their own account regardless of what fee was already deducted at payment time.
- **Funds movement:** checked against the merchant's live token balance (`token_client.balance(&merchant_account)`); panics `InsufficientBalance` if short. Transferred via `MerchantAccountRefundClient::refund(token, amount, payer)` — i.e. the call goes through the merchant's `account` contract, not a direct token transfer from `shade`.
- **Resulting state:** `invoice.amount_refunded += amount_to_refund`; `invoice.status = InvoiceStatus::Refunded`.
- **Accounting fields:** `amount_refunded` is the only field this function updates besides `status`.
- **Event:** `InvoiceRefundedEvent { invoice_id, merchant, amount, timestamp }` — note the `amount` field carries `invoice.amount` (the invoice's face amount at the time of the event, read *after* the update), not the transferred `amount_to_refund` computed earlier in the function. In the ordinary single-shot case these are equal since nothing was refunded before; see [Event reconciliation](#event-reconciliation) for how to reconcile this for callers relying on the event payload.

## Partial refunds — `refund_invoice_partial`

Verified at [`invoice.rs:577-659`](../../contracts/shade/src/components/invoice.rs#L577-L659).

- **Caller:** the invoice's owning merchant (same ownership check as `refund_invoice`).
- **Allowed starting state:** `InvoiceStatus::Paid` or `InvoiceStatus::PartiallyRefunded` — panics `InvalidInvoiceStatus` otherwise. This is what makes repeated partial refunds possible: after one partial refund leaves the invoice `PartiallyRefunded`, the same function can be called again.
- **Refund window:** if `date_paid` is set, enforces the same 7-day (`MAX_REFUND_DURATION`) window as `refund_invoice`. Unlike `refund_invoice`, a missing `date_paid` does **not** panic here — the window check is skipped entirely (`if let Some(date_paid) = ...`).
- **Amount validation:**
  - `amount <= 0` → panics `InvalidAmount`.
  - The core invariant, verified directly in source: `total_refund = invoice.amount_refunded + amount`; if `total_refund > invoice.amount`, panics `InvalidAmount`. So the running sum of every partial (and full) refund ever applied to an invoice can never exceed its `amount` — this is the exact formula enforced.
- **Repeated partial refunds:** allowed by the status check above; each call adds its `amount` to the running `amount_refunded` total, re-validated against the same `<= invoice.amount` invariant every time.
- **Terminal transition:** `new_status = if total_refund == invoice.amount { Refunded } else { PartiallyRefunded }`. Reaching exactly `invoice.amount` in cumulative refunds moves the invoice to the same terminal `Refunded` state `refund_invoice` produces — from that point, further calls to `refund_invoice_partial` panic `InvalidInvoiceStatus` (since `Refunded` is not an allowed starting state).
- **Fee treatment:** identical to `refund_invoice` — the fee taken at payment time is not touched; the full `amount` requested comes out of the merchant's account, checked against the merchant's live balance before transfer.
- **Funds movement:** `MerchantAccountRefundClient::refund(token, amount, payer)`, same as `refund_invoice`.
- **Accounting fields:** `invoice.amount_refunded` set to the new cumulative total; `invoice.status` set per the terminal-transition rule above.
- **Events:** if this call completes the refund (`total_refund == invoice.amount`), emits `InvoiceRefundedEvent { invoice_id, merchant, amount: invoice.amount, timestamp }` (same event as a full refund — this is how an off-chain listener distinguishes "refund completed via partial calls" from "refund completed via `refund_invoice`" only by the preceding `InvoicePartiallyRefundedEvent`s, since the final event itself looks identical). Otherwise emits `InvoicePartiallyRefundedEvent { invoice_id, merchant, amount, total_amount_refunded: total_refund, timestamp }`, where `amount` is this call's increment and `total_amount_refunded` is the running cumulative figure.

## Voids — `void_invoice`

Verified at [`invoice.rs:801-832`](../../contracts/shade/src/components/invoice.rs#L801-L832).

- **Caller:** the invoice's owning merchant.
- **Allowed starting state:** `InvoiceStatus::Pending` only — panics `InvalidInvoiceStatus` for any other status, including `PartiallyPaid`. **A paid invoice (`Paid`, `PartiallyPaid`, `PartiallyRefunded`, or already `Refunded`) cannot be voided** — this is directly enforced by the status check, not inferred.
- **Rationale (interpretation, not stated in code or comments):** voiding is Shade's mechanism for cancelling an invoice that never received funds; once any payment has landed, the refund functions (`refund_invoice`/`refund_invoice_partial`) are the only path back, since they correctly move funds and record `amount_refunded`, which `void_invoice` does not touch.
- **Funds movement:** none — `void_invoice` never reads or transfers via a token client.
- **Resulting state:** `invoice.status = InvoiceStatus::Cancelled`.
- **Terminality:** `Cancelled` is not listed as an allowed starting status for `refund_invoice`, `refund_invoice_partial`, `void_invoice` itself, `pay_invoice`/`pay_invoice_partial` (which require `Pending`/`PartiallyPaid`), or `amend_invoice` (requires `Pending`) — no function read for this page can transition an invoice out of `Cancelled`. Treat it as terminal.
- **Event:** `InvoiceCancelledEvent { invoice_id, merchant, timestamp }`.

## Amendments — `amend_invoice`

Verified at [`invoice.rs:834-884`](../../contracts/shade/src/components/invoice.rs#L834-L884).

- **Caller:** the invoice's owning merchant.
- **Allowed starting state:** `InvoiceStatus::Pending` only — panics `InvalidInvoiceStatus` otherwise. An invoice that has received any payment, been voided, or been refunded cannot be amended.
- **Mutable fields:** `amount` (via `new_amount: Option<i128>`) and `description` (via `new_description: Option<String>`) — both optional; either, both, or neither may be supplied in a single call (`None` leaves the field unchanged).
- **Immutable fields:** `token`, `merchant_id`, `expires_at`, `pricing_mode`, `fiat_pricing` are not parameters to `amend_invoice` and are never written by it — changing the settlement token or expiry requires cancelling (`void_invoice`) and recreating the invoice.
- **Validation:** if `new_amount` is supplied, panics `InvalidAmount` unless it is `> 0`. No re-validation against `admin::calculate_fee` (the "amount must exceed the fee" check `validate_invoice_creation` performs at creation time) is performed on amendment — verified absent from `amend_invoice`'s body.
- **Interaction with paid/refunded amounts:** none possible — since the precondition requires `Pending`, `amount_paid` and `amount_refunded` are always `0` at the time an amendment can occur (nothing has been paid or refunded yet on a still-`Pending` invoice, since payment is the only way to leave `Pending` other than voiding/finalizing).
- **Resulting state:** status unchanged (`Pending`); only `amount` and/or `description` change.
- **Audit/history effects:** none beyond the event below — `amend_invoice` does not write to `DataKey::UserTransactions` / call `history::record_transaction`, and there is no separate amendment-history record; the emitted event is the only on-chain trace of the change.
- **Event:** `InvoiceAmendedEvent { invoice_id, merchant, old_amount, new_amount, timestamp }`, where `old_amount` is the pre-amendment amount and `new_amount` is the post-amendment amount (equal to `old_amount` if the call only changed the description).

## Buyer-initiated expiry claim — `claim_refund`

Not one of the four operations named in the page title, but the only other function in `invoice.rs` that moves funds backward from merchant to buyer, so it is documented here for completeness. Verified at [`invoice.rs:886-951`](../../contracts/shade/src/components/invoice.rs#L886-L951).

- **Caller:** `buyer`, which must equal `invoice.payer` — panics `NotAuthorized` (via the `ContractError` variant) otherwise.
- **Preconditions:** `invoice.expires_at` must be set (panics `InvoiceExpired` if `None`); the current ledger time must be at or past `expires_at` (panics with `EscrowError::EscrowNotExpired` otherwise — note this reuses the `EscrowError` enum, not a dedicated invoice-expiry error); status must not already be `Refunded` (panics `EscrowError::EscrowAlreadyRefunded`); status must be `Paid` or `PartiallyPaid` (panics `InvalidInvoiceStatus` otherwise); `amount_paid - amount_refunded` must be `> 0` (panics `EscrowError::EscrowAlreadyRefunded` if not, i.e. it was already fully refunded through some other path).
- **Amount:** `amount_paid - amount_refunded` — whatever was actually paid in, less anything already refunded. Unlike `refund_invoice`, this is based on `amount_paid`, not `invoice.amount`, so it correctly handles a `PartiallyPaid` invoice that expired before full payment.
- **Funds movement:** same mechanism as the merchant-initiated refunds — `MerchantAccountRefundClient::refund(token, amount, buyer)` from the merchant's account, balance-checked first.
- **Resulting state:** `amount_refunded += amount_to_refund`; `status = Refunded`.
- **Event:** `EscrowExpiredRefundEvent { invoice_id, buyer, amount, token, timestamp }` — a distinct event type from the merchant-initiated refund events, despite producing the same `Refunded` terminal status.

## Event reconciliation

| Operation | Event(s) emitted | Topics/payload | Emission point | State transition represented |
|---|---|---|---|---|
| `refund_invoice` | `InvoiceRefundedEvent` | `invoice_id`, `merchant`, `amount` (invoice's face amount, read post-update), `timestamp` | After the storage write that sets `status = Refunded` | `Paid` → `Refunded` |
| `refund_invoice_partial` (completes the refund) | `InvoiceRefundedEvent` | Same shape as above; `amount` is again `invoice.amount`, not the specific call's increment | After the storage write | `Paid`/`PartiallyRefunded` → `Refunded` |
| `refund_invoice_partial` (does not complete) | `InvoicePartiallyRefundedEvent` | `invoice_id`, `merchant`, `amount` (this call's increment), `total_amount_refunded` (cumulative), `timestamp` | After the storage write | `Paid`/`PartiallyRefunded` → `PartiallyRefunded` |
| `void_invoice` | `InvoiceCancelledEvent` | `invoice_id`, `merchant`, `timestamp` | After the storage write | `Pending` → `Cancelled` |
| `amend_invoice` | `InvoiceAmendedEvent` | `invoice_id`, `merchant`, `old_amount`, `new_amount`, `timestamp` | After the storage write | `Pending` → `Pending` (fields changed) |
| `claim_refund` | `EscrowExpiredRefundEvent` | `invoice_id`, `buyer`, `amount`, `token`, `timestamp` | After the storage write | `Paid`/`PartiallyPaid` → `Refunded` |

**Reconciliation guidance for an off-chain ledger:** because `InvoiceRefundedEvent.amount` always reports the invoice's current face amount rather than the specific transfer that just happened, an off-chain system that wants the actual token amount moved by *that* call should either (a) sum `InvoicePartiallyRefundedEvent.amount` values plus the final completing transfer's implied remainder (`invoice.amount - amount_refunded` immediately before the completing call), or (b) read the invoice's `amount_refunded` field before and after processing the event and take the difference, rather than trusting `InvoiceRefundedEvent.amount` as "the amount just transferred." `claim_refund`'s `EscrowExpiredRefundEvent.amount` does not have this ambiguity — it reports the actual transferred amount directly.

All six events above are defined in [`contracts/shade/src/events.rs`](../../contracts/shade/src/events.rs); see that file for the full `#[contractevent]` struct definitions this table summarizes.

## Related pages

- [Storage model](../architecture/storage-model.md) — `DataKey::Invoice(u64)` and the `Invoice` record's fields (`amount`, `amount_paid`, `amount_refunded`, `status`).
- [Shade contract reference](../contracts/shade.md) — full function list, including `pay_invoice`/`pay_invoice_partial` (how an invoice reaches `Paid`/`PartiallyPaid` in the first place) and the pause/reentrancy guards these functions run under.
- [Merchants](merchants.md) — how `merchant.rs` resolves `merchant_address` to `merchant_id` for the ownership checks used throughout this page.
