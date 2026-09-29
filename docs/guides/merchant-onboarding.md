# Merchant onboarding guide

An ordered, executable checklist for taking a merchant from nothing to accepting payments on Shade, with the required [signer](../glossary.md#admin) and a verification call at every step.

## Prerequisites

- **A funded Stellar account** for the merchant, and a separate funded account for the platform [admin](../glossary.md#admin) if you are also standing up the platform side. See [Prerequisites](../getting-started/prerequisites.md) for toolchain setup and [Quickstart](../getting-started/quickstart.md) for generating and funding test identities with `stellar keys generate` / `stellar keys fund`.
- **A network** (local standalone, futurenet, or testnet) with the `shade` contract already deployed and initialized. See [Quickstart: Steps 1–3](../getting-started/quickstart.md#step-1-configure-network--identities).
- **The Stellar/Soroban CLI**, `stellar`, v23.x. See [Prerequisites](../getting-started/prerequisites.md).
- **The `shade` contract's ID**, referenced below as `$SHADE_ID`.
- **At least one [accepted token](../glossary.md#accepted-token)** already whitelisted by the admin via `add_accepted_token` — this is a prerequisite for step 5 below and for creating any invoice. See [Quickstart: Step 4](../getting-started/quickstart.md#step-4-configure-platform-accounts-tokens-and-fees).
- **Platform admin configuration** already in place: `initialize` has been called, and — if merchant accounts are needed for fund custody — the admin has called `set_account_wasm_hash` so `register_merchant` can deploy the merchant's `account` contract. Without a WASM hash configured, `register_merchant` fails with `WasmHashNotSet` (18).
- **No prerequisite oracle configuration** is required for the steps in this guide. `set_token_oracle` is only needed if the merchant later creates fiat-denominated invoices via `create_fiat_invoice` — out of scope here; see [Fiat pricing and oracles](../concepts/fiat-pricing-and-oracles.md).
- **Merchant information needed up front:** just the merchant's Stellar address. Nothing else is required by `register_merchant` itself — display name, contact info, etc. are not modeled on-chain.

## Ordered onboarding flow

The table below reflects what the contract source actually enforces (see [`contracts/shade/src/components/merchant.rs`](../../contracts/shade/src/components/merchant.rs)), not just the sequence the tracking issue proposed. Two things differ from a strictly linear reading of that list:

- `set_merchant_account`, `set_merchant_key`, and `set_merchant_webhook` do **not** check `is_merchant_active` — they only require the caller to be a registered merchant (`is_merchant`). They can be called even while a merchant is deactivated.
- `set_merchant_accepted_tokens` (and `remove_merchant_accepted_token`) **do** check `is_merchant_active`, and fail with `MerchantNotActive` (32) if the admin has not yet activated the merchant.

That makes `set_merchant_status` (activation) a hard prerequisite for step 5 specifically, not for every step after registration.

| Step | Method | Caller | Required authority | Result |
|---|---|---|---|---|
| 1 | `register_merchant` | Merchant | `merchant.require_auth()` | Merchant is created with `active: true`, `verified: false`; a merchant [account](../glossary.md#merchant-account) contract is deployed and linked. |
| 2 | `set_merchant_status` | Admin | `core::assert_admin` (the stored `DataKey::Admin`) | Merchant's `active` flag is set. New merchants are already `active: true` by default — this step is only needed if an admin previously deactivated the merchant, or to explicitly re-confirm activation. |
| 3 | `verify_merchant` | Admin | `core::assert_admin` | Merchant's `verified` flag is set. This is a separate signal from `active` — nothing in the payment path (`create_invoice`, `pay_invoice`) currently checks `verified`. Treat it as a platform-level trust marker, not a payment gate. |
| 4 | `set_merchant_account` | Merchant | `merchant.require_auth()` | Points `DataKey::MerchantAccount(merchant_id)` at a custody contract address. `register_merchant` already deploys and links one automatically — call this only to repoint the merchant at a different account. |
| 5 | `set_merchant_accepted_tokens` | Merchant | `merchant.require_auth()`, and merchant must be `active` | Sets the merchant's own accepted-token list, validated against the global whitelist. Requires step 2 (active) to have happened first. |
| 6 | `set_merchant_key` | Merchant | `merchant.require_auth()` | Registers a 32-byte ed25519 public key for [signed invoices](../security/signatures.md). |
| 7 | `set_merchant_webhook` | Merchant | `merchant.require_auth()` | Stores a webhook URL string on the merchant record. See [Merchant webhooks and off-chain notifications](webhooks.md) for what this does and does not do. |

Every step after registration authorizes as the **merchant's own address**, except activation and verification, which authorize as the **platform admin**. There is no separate "manager" or "operator" role gating any of these seven calls — [`Role::Manager`/`Role::Operator`](../glossary.md#role) only gates `restrict_merchant_account`, which is not part of this onboarding flow.

### 1. `register_merchant` — mandatory

```rust
fn register_merchant(env: Env, merchant: Address);
```

- **Purpose:** Creates the merchant record and deploys its `account` contract.
- **Signer:** The merchant's own address (`merchant.require_auth()`).
- **CLI:**
  ```bash
  stellar contract invoke \
    --id $SHADE_ID \
    --source merchant \
    --network standalone \
    -- register_merchant \
    --merchant $(stellar keys address merchant)
  ```
- **Expected result:** No return value. Emits `MerchantRegisteredEvent` and `MerchantAccountDeployedEvent`.
- **Verify:**
  ```bash
  stellar contract invoke --id $SHADE_ID --network standalone -- is_merchant --merchant $(stellar keys address merchant)
  # true
  ```
- **Common failure:** `MerchantAlreadyRegistered` (5) if this address already has a `DataKey::MerchantId` entry — check with `is_merchant` first. `WasmHashNotSet` (18) if the admin has not called `set_account_wasm_hash`.
- **Mandatory** — every following step needs a registered merchant ID.

### 2. `set_merchant_status` — optional

> Optional — unlocks: nothing new by itself. Registration already sets `active: true`; this step only matters if the merchant needs to be (re-)activated after an admin deactivated it, and it is a **hard prerequisite for step 5** if that happened.

```rust
fn set_merchant_status(env: Env, admin: Address, merchant_id: u64, status: bool);
```

- **Purpose:** Enable or disable a merchant's ability to create invoices and manage its own accepted-token list.
- **Signer:** The platform admin (`core::assert_admin` against `DataKey::Admin`).
- **CLI:**
  ```bash
  stellar contract invoke \
    --id $SHADE_ID \
    --source admin \
    --network standalone \
    -- set_merchant_status \
    --admin $(stellar keys address admin) \
    --merchant_id 1 \
    --status true
  ```
- **Expected result:** No return value. Emits `MerchantStatusChangedEvent`.
- **Verify:**
  ```bash
  stellar contract invoke --id $SHADE_ID --network standalone -- is_merchant_active --merchant_id 1
  # true
  ```
- **Common failure:** `MerchantNotFound` (6) if `merchant_id` is invalid or exceeds the registered count.

### 3. `verify_merchant` — optional

> Optional — unlocks: a `verified: true` flag readable via `is_merchant_verified`/`get_merchant`. As of the current source, no payment-path function conditions its behavior on this flag — it is a platform trust signal, not a functional gate.

```rust
fn verify_merchant(env: Env, admin: Address, merchant_id: u64, status: bool);
```

- **Purpose:** Marks a merchant as verified by the platform (e.g. after KYB/KYC review conducted off-chain).
- **Signer:** The platform admin.
- **CLI:**
  ```bash
  stellar contract invoke \
    --id $SHADE_ID \
    --source admin \
    --network standalone \
    -- verify_merchant \
    --admin $(stellar keys address admin) \
    --merchant_id 1 \
    --status true
  ```
- **Expected result:** No return value. Emits `MerchantVerifiedEvent`.
- **Verify:**
  ```bash
  stellar contract invoke --id $SHADE_ID --network standalone -- is_merchant_verified --merchant_id 1
  # true
  ```
- **Common failure:** `MerchantNotFound` (6) via the underlying `get_merchant` lookup.

### 4. `set_merchant_account` — optional

> Optional — unlocks: repointing which `account` contract custodies this merchant's funds. `register_merchant` already deploys and links a default account, so most merchants never need to call this directly.

```rust
fn set_merchant_account(env: Env, merchant: Address, account: Address);
```

- **Purpose:** Sets `DataKey::MerchantAccount(merchant_id)` to a specific `account` contract address.
- **Signer:** The merchant's own address.
- **CLI:**
  ```bash
  stellar contract invoke \
    --id $SHADE_ID \
    --source merchant \
    --network standalone \
    -- set_merchant_account \
    --merchant $(stellar keys address merchant) \
    --account $ACCOUNT_CONTRACT_ID
  ```
- **Expected result:** No return value. **No event is emitted** for this call — the source has no `events::publish_*` call in `set_merchant_account`.
- **Verify:**
  ```bash
  stellar contract invoke --id $SHADE_ID --network standalone -- get_merchant_account --merchant_id 1
  ```
- **Common failure:** `MerchantNotFound` (6) if the caller is not a registered merchant.

### 5. `set_merchant_accepted_tokens` — optional but recommended

> Optional — unlocks: restricting which tokens this specific merchant accepts, out of the globally whitelisted set. If skipped, [`is_token_accepted_for_merchant`](../../contracts/shade/src/components/merchant.rs#L444-L460) falls back to accepting **every** globally whitelisted token for this merchant — so skipping this step is not a blocker, but it does mean the merchant accepts the whole global list by default.

```rust
fn set_merchant_accepted_tokens(env: Env, merchant: Address, tokens: Vec<Address>);
```

- **Purpose:** Narrows the merchant's own accepted-token list.
- **Signer:** The merchant's own address, and the merchant **must be active** (see the flow note above).
- **CLI:**
  ```bash
  stellar contract invoke \
    --id $SHADE_ID \
    --source merchant \
    --network standalone \
    -- set_merchant_accepted_tokens \
    --merchant $(stellar keys address merchant) \
    --tokens '["'$TOKEN_ID'"]'
  ```
- **Expected result:** No return value. Emits `MerchantTokensSetEvent`.
- **Verify:**
  ```bash
  stellar contract invoke --id $SHADE_ID --network standalone -- get_merchant_accepted_tokens --merchant $(stellar keys address merchant)
  ```
- **Common failures:** `MerchantNotActive` (32) if step 2 has not run; `TokenNotAccepted` (12) if a supplied token is not on the global whitelist (admin must call `add_accepted_token` first).

### 6. `set_merchant_key` — optional

> Optional — unlocks: `create_invoice_signed`, i.e. creating invoices from an off-chain-signed payload without the merchant submitting the Soroban transaction itself. Skip this if the merchant will only ever call `create_invoice` directly.

```rust
fn set_merchant_key(env: Env, merchant: Address, key: BytesN<32>);
```

- **Purpose:** Registers a 32-byte ed25519 public key used to verify signed invoice payloads.
- **Signer:** The merchant's own address.
- **CLI:**
  ```bash
  stellar contract invoke \
    --id $SHADE_ID \
    --source merchant \
    --network standalone \
    -- set_merchant_key \
    --merchant $(stellar keys address merchant) \
    --key $MERCHANT_PUBLIC_KEY_HEX
  ```
- **Expected result:** No return value. Emits `MerchantKeySetEvent`.
- **Verify:**
  ```bash
  stellar contract invoke --id $SHADE_ID --network standalone -- get_merchant_key --merchant $(stellar keys address merchant)
  ```
- **Common failure:** `MerchantNotFound` (6) if the caller is not a registered merchant. See [Signed invoices](../security/signatures.md) for the full signing scheme and message construction.

### 7. `set_merchant_webhook` — optional

> Optional — unlocks: an on-chain record of a webhook destination that an off-chain notifier service can read and deliver payment/event notifications to. The `shade` contract itself never makes an HTTP call — see [Merchant webhooks and off-chain notifications](webhooks.md) for the full architecture.

```rust
fn set_merchant_webhook(env: Env, merchant: Address, webhook: String);
```

- **Purpose:** Stores a webhook URL string on the merchant record.
- **Signer:** The merchant's own address.
- **CLI:**
  ```bash
  stellar contract invoke \
    --id $SHADE_ID \
    --source merchant \
    --network standalone \
    -- set_merchant_webhook \
    --merchant $(stellar keys address merchant) \
    --webhook "https://merchant.example.com/webhooks/shade"
  ```
- **Expected result:** No return value. Emits `MerchantWebhookSetEvent` (which carries the full webhook string in its payload — see [Webhooks: on-chain configuration](webhooks.md#on-chain-webhook-configuration) for what this means for anyone reading the event stream).
- **Verify:**
  ```bash
  stellar contract invoke --id $SHADE_ID --network standalone -- get_merchant_webhook --merchant_id 1
  ```
- **Common failure:** `MerchantNotFound` (6) if the caller is not a registered merchant.

> **Warning:** `set_merchant_webhook` performs **no URL validation** — no scheme check, no length limit, no reachability check. The contract stores whatever string is supplied. Do not assume "the URL passed on-chain" implies it is a well-formed HTTPS URL; the off-chain notifier that reads it must validate it itself. See [SSRF](webhooks.md#ssrf-considerations-for-the-notifier).

## Verification

Every write above has a corresponding read, tabulated here for quick reference:

| Write | Read/verification |
|---|---|
| `register_merchant` | `is_merchant`, `find_merchant_id`, `get_merchant` |
| `set_merchant_status` | `is_merchant_active` |
| `verify_merchant` | `is_merchant_verified` |
| `set_merchant_account` | `get_merchant_account` |
| `set_merchant_accepted_tokens` | `get_merchant_accepted_tokens`, `is_token_accepted_for_merchant` |
| `set_merchant_key` | `get_merchant_key` |
| `set_merchant_webhook` | `get_merchant_webhook` |

## Post-onboarding smoke test

Once a merchant has completed at least step 1 (registration) and step 2 (active), and has at least one accepted token, confirm the whole path works end to end using the same pattern as [Quickstart: Steps 6–7](../getting-started/quickstart.md#step-6-create-an-invoice):

1. **Create an invoice:**
   ```bash
   INVOICE_ID=$(stellar contract invoke \
     --id $SHADE_ID --source merchant --network standalone \
     -- create_invoice \
     --merchant $(stellar keys address merchant) \
     --description "Smoke test invoice" \
     --amount 1000000 \
     --token $TOKEN_ID \
     --expires_at 0)
   ```
2. **Pay it with a test payer:**
   ```bash
   stellar contract invoke \
     --id $SHADE_ID --source payer --network standalone \
     -- pay_invoice \
     --payer $(stellar keys address payer) \
     --invoice_id $INVOICE_ID
   ```
3. **Confirm settlement:**
   ```bash
   stellar contract invoke --id $SHADE_ID --network standalone -- get_invoice --invoice_id $INVOICE_ID
   # status: "Paid"
   ```
4. **Confirm merchant/invoice state:** re-run `get_invoice` and check `amount_paid` equals the invoice amount and `payer` matches the test payer's address.
5. **Confirm analytics updated:**
   ```bash
   stellar contract invoke --id $SHADE_ID --network standalone -- get_merchant_analytics --merchant $(stellar keys address merchant) --token $TOKEN_ID
   # transaction_count and total_volume reflect the smoke-test payment
   ```
6. **Confirm the events fired** (optional, useful when validating an indexer — see [Event indexing guide](indexing-events.md)): `InvoiceCreatedEvent` on step 1 and `InvoicePaidEvent` plus `PlatformFeeRoutedEvent` on step 2.

The repository does not currently ship a single scripted smoke-test command for this flow — the steps above are the manual CLI sequence. If a scripted integration harness is added later (see the crate-level `integration_test.rs`/`test_integration.rs` files under `contracts/*/src/`, which cover contract-internal flows but not this cross-cutting onboarding path), link it from here instead of repeating these steps.

## Troubleshooting

| Symptom | Likely cause | Verification | Fix |
|---|---|---|---|
| `create_invoice` fails right after registration | Merchant is not active (activation defaults to `true` at registration, but may have been explicitly deactivated) | `is_merchant_active --merchant_id <id>` | Admin calls `set_merchant_status(admin, merchant_id, true)` |
| `verify_merchant` was called but payments still behave the same | Expected — `verified` is not currently checked by any payment-path function | `is_merchant_verified --merchant_id <id>` | No fix needed; this is not a functional gate today |
| `set_merchant_accepted_tokens` fails with `TokenNotAccepted` | The token isn't on the *global* whitelist yet | `is_accepted_token --token <address>` | Admin calls `add_accepted_token` first |
| `set_merchant_accepted_tokens` fails with `MerchantNotActive` | Merchant was never activated, or was deactivated | `is_merchant_active --merchant_id <id>` | Admin calls `set_merchant_status(admin, merchant_id, true)` |
| `pay_invoice` fails with `TokenNotAcceptedByMerchant` | Payer used a token the merchant's own list excludes (even though it's globally whitelisted) | `is_token_accepted_for_merchant --merchant <address> --token <address>` | Merchant adds the token via `set_merchant_accepted_tokens`, or payer uses a token the merchant already accepts |
| `get_merchant_account` panics with `MerchantAccountNotSet` | No account was ever linked — should not happen after `register_merchant`, but can if the storage was somehow never written | `get_merchant_account --merchant_id <id>` | Merchant calls `set_merchant_account` with a valid deployed `account` contract address |
| `register_merchant` fails with `WasmHashNotSet` | Admin never called `set_account_wasm_hash` | Attempt `register_merchant` and read the panic code | Admin calls `set_account_wasm_hash` with a valid installed WASM hash before any merchant registers |

See [Errors reference](../reference/errors.md) for the full list of `shade` error codes and their corrective actions.

## Related pages

- [Errors reference](../reference/errors.md) — every error code referenced above.
- [Merchant webhooks and off-chain notifications](webhooks.md) — what `set_merchant_webhook` does and does not guarantee.
- [Event indexing guide](indexing-events.md) — building a reliable index of the events emitted throughout this flow.
- [Signed invoices](../security/signatures.md) — the full scheme unlocked by `set_merchant_key`.
- [Quickstart tutorial](../getting-started/quickstart.md) — the same flow with full network setup from scratch.

← [Back to guides](README.md)
