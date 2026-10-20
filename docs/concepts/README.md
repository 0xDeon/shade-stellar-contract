# Concepts

Deep dives into individual mechanisms, written from the [concept template](../contributing/templates/concept-template.md). Reference pages and guides link here for the "why," not just the "how."

- [Invoice lifecycle and statuses](./invoice-lifecycle.md)
- [Payment flows: full, partial, and batch](./payments.md) — preconditions, fee splitting, rounding, and batch atomicity for `pay_invoice`, `pay_invoice_partial`, and `pay_invoices_batch`.
- [Escrow](./escrow.md)
- [Subscriptions and recurring billing](./subscriptions.md)
- [Subscription billing authorization and allowances](./subscription-billing-authorization.md) — `authorize_billing`, `get_billing_allowance`, `revoke_billing_authorization`, and how plan price changes affect an existing allowance (standalone `subscription` contract).
- [Subscription grace periods and past-due handling](./subscription-grace-periods.md) — the `Active`/`PastDue`/`Terminated` state machine, recovery, and event emission (standalone `subscription` contract).
- Invoices, drafts, and signed invoices — *planned*.
- [Refunds and voids](./refunds-and-voids.md) — full refunds, partial refunds, voids, amendments, and buyer-initiated expiry claims.
- [Fees, volume discounts, and time-locked fee changes](./fees.md) — `calculate_fee`, merchant volume discounts, and the `propose_fee` / `execute_fee` timelock.
- Escrow and arbiter release — *planned*.
- Subscriptions and recurring billing — *planned*.
- [Merchants](./merchants.md) — registration, activation, verification, configuration, and the state-to-operation matrix.
- [Event ticketing, dynamic pricing, and resale](./event-ticketing.md)
- [Event waitlists](./event-waitlists.md) — `join_waitlist`, `get_waitlist`, FIFO positions, and automatic promotion on refund (standalone `ticketing` contract).
- [Merchant analytics and transaction history](./analytics-and-history.md)
- [Token analytics and market share metrics](./token-analytics.md) — protocol-wide token volume, dominance, and basis-point market share.
- [Amounts, token decimals, and rounding](./amounts-and-rounding.md) — base units, per-path rounding direction, remainders, and conversion helpers.
- [Choosing a subscription model](./choosing-a-subscription-model.md) — hub subscriptions vs. the standalone `subscription` contract.
- [Auto-withdrawal and merchant settlement](./auto-withdrawal.md)
- [Payment payloads, swap routing, and the cross-chain bridge placeholder](./payment-payloads-and-routing.md)
- Fiat-pegged campaign goals and oracle pricing — *planned*.
- Creator fund vesting — *planned*.
- Dynamic hard-cap voting — *planned*.
- Stretch goals — *planned*.
- DAO governance and contract upgrades — *planned*.
- Cross-chain bridge deposits and pledges — *planned*.
- Reentrancy guard — *planned*.

← [Back to documentation home](../README.md)
