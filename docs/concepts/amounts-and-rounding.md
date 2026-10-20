# Amounts, token decimals, and rounding

Every amount in Shade is an `i128` count of a token's base units. This page defines base units versus display units, the unit and rounding direction of every arithmetic path, who keeps each remainder, and the overflow and negative-amount guards. It is for integrators who need their numbers to match the contract to the last stroop.

## Base units and display units

A token contract stores integers. Its `decimals()` function says how many of the lowest digits are fractional.

```
display_amount = base_units / 10^decimals
base_units     = display_amount * 10^decimals
```

The contract never converts between the two. Every `amount`, `fee`, `total_volume`, and balance you pass or read is in base units. Convert only at the edge of your application.

| Asset | Contract | Decimals | 1 display unit in base units |
|---|---|---|---|
| XLM | Stellar Asset Contract (SAC) | 7 | `10_000_000` (1 stroop = `1`) |
| USDC (Stellar classic asset) | SAC | 7 | `10_000_000` |
| A custom Soroban token | Its own contract | Whatever it returns | `10^decimals` |

Classic Stellar assets wrapped by a SAC use 7 decimals. A custom token can use any value, so read it instead of assuming.

> **Warning:** Never hard-code 7. Shade accepts any whitelisted token, and the fee-discount thresholds, analytics, and oracle math all depend on the decimals of the token in question. A 6-decimal token and a 7-decimal token with the same display amount differ by a factor of 10 in base units.

### Reading decimals

On-chain, call the token contract:

```rust
use soroban_sdk::{token, Address, Env};

fn token_decimals(env: &Env, token: &Address) -> u32 {
    token::TokenClient::new(env, token).decimals()
}
```

Off-chain, simulate a call to `decimals` on the token contract. This example is illustrative and uses `@stellar/stellar-sdk`:

```ts
import { Contract, Networks, TransactionBuilder, rpc, scValToNative } from "@stellar/stellar-sdk";

async function readDecimals(server: rpc.Server, sourcePublicKey: string, tokenId: string): Promise<number> {
  const source = await server.getAccount(sourcePublicKey);
  const tx = new TransactionBuilder(source, { fee: "100", networkPassphrase: Networks.TESTNET })
    .addOperation(new Contract(tokenId).call("decimals"))
    .setTimeout(30)
    .build();
  const sim = await server.simulateTransaction(tx);
  if (!rpc.Api.isSimulationSuccess(sim) || !sim.result) throw new Error("decimals simulation failed");
  return Number(scValToNative(sim.result.retval));
}
```

Cache the result per token. It does not change.

## Rounding rules by path

All Shade arithmetic is integer arithmetic on `i128`. Division truncates toward zero, and every operand is positive on a valid path, so truncation is **floor**. No path uses banker's rounding or rounds to nearest. The one place that rounds up is the fiat-goal conversion from fiat to tokens.

| Path | Unit | Formula | Rounds | Who keeps the remainder |
|---|---|---|---|---|
| Platform fee (`calculate_fee`, `compute_split`) | basis points | `amount * fee_bps / 10_000` | Down | Merchant |
| Volume discount on the fee rate | percent of `fee_bps` | `fee_bps * (100 - pct) / 100` | Down (rate) | Payer side: lower rate, lower fee |
| Merchant share of a payment | base units | `amount - platform_fee` | Exact | Merchant gets the rest |
| Partial payment | base units | caller supplies `amount`, fee floored per payment | Down, per payment | Merchant |
| Fiat invoice quote | token base units | `fiat * 10^token_dec * 10^price_dec / (price * 10^fiat_dec)` | Down | Merchant is short by under 1 base unit |
| Fiat goal: fiat to tokens | token base units | same inputs, ceiling division | **Up** | Contributor overpays by under 1 base unit |
| Fiat goal: tokens to fiat | fiat minor units | `tokens * price * 10^fiat_dec / (10^token_dec * 10^price_dec)` | Down | Contributor |
| Partial invoice refund | base units | caller supplies `amount` | None | Not applicable |
| Dynamic ticket price, early bird | base units | `base - floor(base * discount_bps / 10_000)` | Price rounds **up** | Organizer |
| Dynamic ticket price, late markup | base units | `base + floor(base * markup_bps / 10_000)` | Markup rounds down | Buyer |
| Bulk ticket group discount | base units | `gross - floor(gross * discount_bps / 10_000)` | Total rounds **up** | Merchant |
| Resale royalty | base units | `floor(price * royalty_bps / 10_000)`, seller gets `price - royalty` | Royalty down | Seller |
| Prorated subscription refund (standalone) | base units | `amount * remaining_seconds / interval` | Down | Merchant or `creator` |
| Upgrade cost (standalone) | base units | `new_amount - prorated_refund(old)`, floored at 0 | Refund down, so cost up | Merchant or `creator` |
| Token market share | basis points | `token_volume * 10_000 / total_volume` | Down | Not distributed |

### Fees

`set_fee(env, admin, token, fee)` and `propose_fee`/`execute_fee` store the fee in **basis points** as `i128`. `300` is 3%, and `10_000` is 100%. The amount is computed by `calculate_fee` in [`contracts/shade/src/components/admin.rs#L195-L206`](../../contracts/shade/src/components/admin.rs#L195-L206):

```
fee = amount * fee_bps / 10_000     // floor
```

A non-positive `amount` returns `0`. The payment path in [`platform_fee.rs#L42-L70`](../../contracts/shade/src/components/platform_fee.rs#L42-L70) panics with `InvalidAmount` instead, and also panics if `fee >= amount`. Invoice creation rejects any invoice where `amount <= fee`.

Resolution order, discount tiers, and worked examples are in [Fees](./fees.md).

> **Warning:** The volume discount thresholds are raw base units of gross volume per merchant and token: `10_000`, `50_000`, and `200_000`. For a 7-decimal token, `10_000` is only 0.001 of a display unit, so the first tier is reached almost immediately. For a 0-decimal token it is 10,000 whole units.

> **Warning:** `set_fee` and `propose_fee` do not range-check `fee`. Only `set_merchant_platform_fee` enforces `0..=10_000`. Admins must keep fees within `0..=10_000`.

### Fiat conversion and oracle decimals

`resolve_fiat_invoice_amount` in [`contracts/shade/src/components/invoice.rs#L33-L57`](../../contracts/shade/src/components/invoice.rs#L33-L57) converts a fiat price to token base units with three decimal settings:

| Value | Meaning | Set by |
|---|---|---|
| `fiat_pricing.amount` | Fiat in **minor units**, scaled by `fiat_pricing.decimals` (`1_000_00` with `decimals = 2` is 1,000.00) | Merchant, at `create_fiat_invoice` |
| `price` | Price of **one whole token** in the quote currency, scaled by `price_decimals` | Oracle `get_price(token, currency)` |
| `token_decimals`, `price_decimals` | Stored in `OracleConfig` | Admin, at `set_token_oracle` |

```
token_base_units = fiat_amount * 10^token_decimals * 10^price_decimals
                   / (price * 10^fiat_decimals)
```

The result floors, and a result of `0` or a non-positive `price` panics with `OraclePriceUnavailable`. The quote refreshes on each payment only while `amount_paid == 0`. After the first payment the stored `amount` is frozen. See [Fiat pricing and oracles](./fiat-pricing-and-oracles.md) for staleness and downtime behavior.

> **Warning:** `token_decimals` in `OracleConfig` is configured by the admin, not read from the token. A wrong value silently scales every quote by a power of ten. Verify it against `decimals()` when you register an oracle.

The fiat-goal component in [`fiat_goals.rs#L74-L153`](../../contracts/shade/src/components/fiat_goals.rs#L74-L153) uses the same inputs but rounds fiat-to-token **up** so a contribution always covers the fiat target, and caps `fiat_decimals` at 18. Its multiplications are checked and fail with a contract error.

### Partial payments

`pay_invoice_partial(payer, invoice_id, amount)` accepts any `amount > 0` with `amount_paid + amount <= invoice.amount`. The fee is computed on each payment separately, so no remainder is carried between payments. The last payment, or `pay_invoice` for the remaining balance, brings `amount_paid` exactly to `amount`.

Because `floor(a) + floor(b) <= floor(a + b)`, paying in parts can yield a slightly lower total fee than one payment of the full amount. The difference stays with the merchant. Example at 300 bps: paying 4,999 once costs a fee of `149`. Paying it as ten payments of 499 and one of 9 costs `10 * 14 + 0 = 140`, because `499 * 300 / 10_000` floors from 14.97 to 14 and `9 * 300 / 10_000` floors to 0.

### Refunds

| Operation | Amount | Fee handling |
|---|---|---|
| `refund_invoice` | `invoice.amount - invoice.amount_refunded` | The merchant account sends the full remainder to the payer. |
| `refund_invoice_partial` | Caller-supplied `amount > 0`, cumulative total at most `invoice.amount` | Same. No proration. |

Refunds come out of the merchant account balance. No code path returns the platform fee that was taken at payment time, and refunds never decrement analytics. Treat the platform fee as non-refundable in your accounting.

### Splits and prorating

A payment split into `platform_fee` and `merchant_amount` always satisfies `platform_fee + merchant_amount == amount`. Nothing is created or lost, and the floor puts the remainder on the merchant side.

The standalone `subscription` contract prorates with `amount * remaining / interval`, using `checked_mul` and returning `0` on overflow. See [Subscription contract reference](../contracts/subscription.md#refunds). The party paying the refund never owes a fractional unit, because the result floors.

## InvoiceFilter bounds are `u128`

`InvoiceFilter.min_amount` and `max_amount` are `Option<u128>`, while `Invoice.amount` is `i128` ([`types.rs#L398-L431`](../../contracts/shade/src/types.rs#L398-L431)). The filter code casts the bounds with `as i128` before comparing ([`invoice.rs#L548-L557`](../../contracts/shade/src/components/invoice.rs#L548-L557), [`search.rs#L117-L125`](../../contracts/shade/src/components/search.rs#L117-L125)).

- A valid invoice amount is always positive, because creation rejects `amount <= 0`. An unsigned bound can only express meaningful values, and a caller cannot pass a negative bound.
- The source records no other rationale for the unsigned type.
- The cast is unchecked. A bound above `i128::MAX` wraps to a negative number. A wrapped `min_amount` matches every invoice, and a wrapped `max_amount` matches none. Keep bounds at or below `i128::MAX`.

## Overflow and negative-amount protections

| Protection | Where |
|---|---|
| Release builds trap on integer overflow instead of wrapping | `overflow-checks = true` in [`Cargo.toml#L10-L12`](../../Cargo.toml#L10-L12) |
| `checked_mul` before dividing by 10,000 | `bps_of` in [`event.rs#L424-L429`](../../contracts/shade/src/components/event.rs#L424-L429), and the fiat-goal `mul` |
| `saturating_add` and `saturating_mul` on counters and totals | Ticket sales, subscription timing, pledge totals |
| Positive-amount checks | Invoice creation, `pay_invoice_partial`, `refund_invoice_partial`, `compute_split`, subscription plan creation, resale price, and `create_fiat_invoice` |
| Fee must leave something for the merchant | Invoice creation (`amount > fee`), `compute_split` (`fee < amount`) |

> **Note:** Plain `*` in `resolve_fiat_invoice_amount` traps on overflow with no contract error code, because `overflow-checks` is on. Keep fiat amounts and decimals within `i128` range after scaling.

## Conversion helpers

### Rust

These helpers mirror the contract formulas. They return `None` on overflow.

```rust
fn pow10(decimals: u32) -> Option<i128> {
    10i128.checked_pow(decimals)
}

/// Platform fee: floor(amount * fee_bps / 10_000).
fn fee_floor(amount: i128, fee_bps: i128) -> Option<i128> {
    if amount <= 0 || fee_bps == 0 {
        return Some(0);
    }
    amount.checked_mul(fee_bps).map(|v| v / 10_000)
}

/// Fiat minor units to token base units, floor. Matches resolve_fiat_invoice_amount.
fn fiat_to_token_floor(
    fiat: i128,
    fiat_decimals: u32,
    price: i128,
    price_decimals: u32,
    token_decimals: u32,
) -> Option<i128> {
    let num = fiat
        .checked_mul(pow10(token_decimals)?)?
        .checked_mul(pow10(price_decimals)?)?;
    let den = price.checked_mul(pow10(fiat_decimals)?)?;
    if den <= 0 { return None; }
    Some(num / den)
}

/// Prorated amount, floor. Matches the standalone subscription refund.
fn prorate_floor(amount: i128, remaining: u64, interval: u64) -> Option<i128> {
    if interval == 0 { return Some(0); }
    amount.checked_mul(remaining as i128).map(|v| v / interval as i128)
}
```

### TypeScript

Use `BigInt` throughout. `number` loses precision above 2^53, which a 7-decimal token reaches at about 900 million display units.

```ts
const pow10 = (d: number): bigint => 10n ** BigInt(d);

/** "12.5" with decimals 7 -> 125000000n. Rejects extra fractional digits instead of rounding. */
export function toBaseUnits(display: string, decimals: number): bigint {
  const m = /^(\d+)(?:\.(\d*))?$/.exec(display.trim());
  if (!m) throw new Error("invalid amount");
  const frac = m[2] ?? "";
  if (frac.length > decimals) throw new Error("too many decimal places");
  return BigInt(m[1]) * pow10(decimals) + BigInt(frac.padEnd(decimals, "0") || "0");
}

/** 125000000n with decimals 7 -> "12.5". */
export function fromBaseUnits(units: bigint, decimals: number): string {
  const neg = units < 0n;
  const abs = neg ? -units : units;
  const whole = abs / pow10(decimals);
  const frac = (abs % pow10(decimals)).toString().padStart(decimals, "0").replace(/0+$/, "");
  return `${neg ? "-" : ""}${whole}${frac ? "." + frac : ""}`;
}

/** Platform fee: floor(amount * feeBps / 10000). BigInt division truncates toward zero. */
export const feeFloor = (amount: bigint, feeBps: bigint): bigint =>
  amount <= 0n ? 0n : (amount * feeBps) / 10_000n;

/** Share in basis points, floor, matching get_token_market_share. */
export const shareBps = (volume: bigint, total: bigint): bigint =>
  total === 0n ? 0n : (volume * 10_000n) / total;
```

Use `fromBaseUnits` only for display. Send `bigint` values to the contract as `i128` in your SDK's `ScVal` conversion.

## Related pages

- [Fees, volume discounts, and time-locked fee changes](./fees.md)
- [Payment flows: full, partial, and batch](./payments.md)
- [Fiat pricing and oracles](./fiat-pricing-and-oracles.md)
- [Refunds and voids](./refunds-and-voids.md)
- [Token analytics and market share metrics](./token-analytics.md)
- [Subscription contract reference](../contracts/subscription.md)
