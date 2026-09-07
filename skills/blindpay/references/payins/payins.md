# Payins

Create a payin, deliver funds to the destination, and track it through settlement.

Source: https://blindpay.com/docs/payins

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

Receive covers the fiat-in side of BlindPay: a customer sends a bank transfer (ACH, wire, Pix, SPEI, Transfers, or PSE), and BlindPay converts it to stablecoins and credits the customer's wallet or managed balance. This section is for teams collecting money from customers, such as onramps, remittance apps, or marketplaces paying in fiat.

## How it works

Every payin starts with a quote that locks the payment method, amount, fee split, and destination for 5 minutes. Create the payin within that window.

```
payin quote -> payin (create within 5 min) -> BlindPay detects the deposit -> settled
```

1. Create a payin quote. It returns the payment instructions to hand to the sender.
2. Create the payin, referencing the quote.
3. BlindPay watches for the deposit, converts it, and sends stablecoins to the customer.

**Note:**

On development instances every payin completes automatically about 30 seconds after initiation.

### Payment methods

| `payment_method` | Currency | What the sender sees |
| --- | --- | --- |
| `ach` | USD | `memo_code` + `blindpay_bank_details` |
| `wire` | USD | `memo_code` + `blindpay_bank_details` |
| `rtp` | USD | `memo_code` + `blindpay_bank_details` (always the memo-code path, even with a virtual account) |
| `international_swift` | USD | `blindpay_bank_details` (requires an approved virtual account, see below) |
| `pix` | BRL | `pix_code` (copyable text or QR code) |
| `ted` | BRL | `memo_code` + `blindpay_bank_details` (requires the `ted_payin` subscription feature, see below) |
| `spei` | MXN | CLABE number |
| `transfers` | ARS | CBU number |
| `pse` | COP | payment link |

For `ach`/`wire`, the memo code is shown only when the customer has no enabled virtual account; with one, BlindPay displays dedicated account details instead and ignores the memo code. See [virtual accounts](../virtual-accounts/virtual-accounts.md).

`rtp` is the exception: no virtual account carries an RTP rail, so an `rtp` payin always shows the shared memo-code account, even when the customer has an approved virtual account for `ach`/`wire`.

`international_swift` payins are only available to receivers with an approved virtual account; there is no memo-code fallback for SWIFT.

`ted` is a Brazilian wire-style rail, gated behind the `ted_payin` subscription feature. Creating a `ted` payin without it enabled fails with 400. Contact BlindPay to enable it for your instance.

## Cover fees

Fees can be paid by either party. Set `cover_fees` on the payin quote:

| `cover_fees` | Who pays | Fee is deducted from |
| --- | --- | --- |
| `false` | The customer | The stablecoin amount the customer receives (most common) |
| `true` | The sender | The fiat amount the sender sends, added on top |

![Payin quote with cover_fees false: the customer pays the fees](https://pub-4fabf5dd55154f19a0384b16f2b816d9.r2.dev/blindpay-payin-quote-cover-fees-false-min.jpg)

![Payin quote with cover_fees true: the sender pays the fees](https://pub-4fabf5dd55154f19a0384b16f2b816d9.r2.dev/blindpay-payin-quote-cover-fees-true-min.jpg)

A payin is the object that actually moves money: it consumes a payin quote and tells BlindPay to start waiting for the customer's deposit. Everything about the payment (amount, method, fees, destination) was already locked in when you created the [payin quote](payin-quotes.md); the payin itself takes a single field.

## How it works

```
payin quote (pq_...) -> create the payin -> display instructions to the payer -> BlindPay detects the deposit -> settles
```

A payin quote expires 5 minutes after creation, so create the payin before then. Once created, a payin cannot be canceled: if the payer never sends the money, the payin simply stays `processing` until it is cleaned up on BlindPay's side.

**Before you start:**

1. Create an account at https://app.blindpay.com/sign-up
2. Create a development instance (see essentials/instances.md)
3. Create your API key (see essentials/api-keys.md)

You also need a customer with a blockchain wallet or a managed wallet, and a payin quote (`pq_...`).

**Advanced flavor (stablecoin and blockchain mechanics)**

A payin is BlindPay's on-ramp: fiat comes in from a sender, stablecoins go out to a wallet. This page is the payin reference: the flow, the destinations, statuses, testing, and webhooks. The step-by-step flows live in the per-destination tutorials.

## How it works

1. Create a [payin quote](payin-quotes.md) for the amount, payment method, and destination. The quote locks the exchange rate, the fees, and generates the payment instructions to hand to the sender.
2. Create the payin using the quote ID. This starts BlindPay watching for the deposit to arrive.
3. The sender completes the transfer using the payment instructions (a bank wire, a Pix code, a CLABE, a CBU, or a payment link, depending on the method).
4. Once the fiat lands, BlindPay converts it and delivers the equivalent stablecoins to the destination wallet, then fires a `payin.complete` webhook.

A payin quote expires 5 minutes after creation, so create the payin before then. Once created, a payin cannot be canceled: if the sender never completes the deposit, the payin stays `processing` until it is cleaned up on BlindPay's side.

The destination is set on the payin quote and is one of:

| Field | Wallet type | Custody | Tutorial |
| --- | --- | --- | --- |
| `wallet_id` (`bl_...`) | Managed wallet | BlindPay-custodied | [Payin with managed wallet](payin-managed-wallet.md) |
| `blockchain_wallet_id` (`bw_...`) | External blockchain wallet | Customer-controlled | [Payin with blockchain wallet](payin-blockchain-wallet.md) |

A payin quote never targets a bank account (`ba_...`); that identifier belongs to the payout side.

**Before you start:**

1. Create an account at https://app.blindpay.com/sign-up
2. Create a development instance (see essentials/instances.md)
3. Create your API key (see essentials/api-keys.md)

You also need a customer with a [managed wallet](../wallets/wallets.md) (`bl_...`) or a [blockchain wallet](blockchain-wallets.md) (`bw_...`), and a [payin quote](payin-quotes.md) (`pq_...`).

## Create a payin

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID.

Replace `pq_000000000000` with the payin quote you generated previously.

```bash [cURL]
curl https://api.blindpay.com/v1/instances/in_000000000000/payins/evm \
  --request POST \
  --header 'Content-Type: application/json' \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --data '{
  "payin_quote_id": "pq_000000000000"
}'
```

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

**Note:**

The endpoint path is `/payins/evm` for every payment method, including Pix, SPEI, Transfers, and PSE. The name is historical; it does not restrict which rail or network the payin uses.

**Advanced flavor (stablecoin and blockchain mechanics)**

**Note:**

The endpoint path is `/payins/evm` regardless of the destination network. The name is historical; it does not restrict the payin to EVM chains, and it works identically whether the payin quote's destination is a Stellar, Solana, or EVM wallet.

### Response

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

The response carries the payment instructions for whichever `payment_method` was set on the quote. Only the field relevant to that method is populated; the rest are `null`.

```json
{
  "id": "pi_000000000000",
  "status": "processing",
  "pix_code": null,
  "memo_code": "8K45GHBNT6BQ6462",
  "clabe": null,
  "partner_fee": 0,
  "customer_id": "re_000000000000",
  "receiver_amount": 9900,
  "payment_method": "ach",
  "sender_amount": 10000,
  "billing_fee_amount": null,
  "transaction_fee_amount": 100,
  "blindpay_bank_details": {
    "routing_number": "021000089",
    "account_number": "31254097",
    "account_type": "Business checking",
    "beneficiary": {
      "name": "Example Beneficiary, Inc.",
      "address_line_1": "1160 Battery St. East, Suite 100",
      "address_line_2": "San Francisco, CA 94111"
    },
    "receiving_bank": {
      "name": "Example Bank, N.A.",
      "address_line_1": "399 Park Avenue",
      "address_line_2": "New York, NY 10043"
    }
  },
  "tracking_transaction": { "step": "processing" },
  "tracking_payment": { "step": "on_hold" },
  "tracking_complete": { "step": "on_hold" },
  "tracking_partner_fee": { "step": "on_hold" }
}
```

`blindpay_bank_details` is derived from which funding path the payin uses, and only one of these ever applies at once:

| Condition | `blindpay_bank_details` shows |
| --- | --- |
| `memo_code` is set (no approved virtual account) | BlindPay's shared memo-code account, shown above |
| Customer has an approved virtual account | That virtual account's own dedicated routing and account number |
| Neither (fallback) | BlindPay's default account (routing `121145349`, beneficiary BlindPay, Inc.) |

Read `blindpay_bank_details` together with whether `memo_code` is null to know which one you're getting.

The virtual account behind `blindpay_bank_details` is the exact one the deposit landed on. For payins created before an instance supported multiple virtual accounts per customer, BlindPay falls back to the customer's oldest approved virtual account, which is safe since that era only ever allowed one.

**Advanced flavor (stablecoin and blockchain mechanics)**

`receiver_amount` is the stablecoin amount, in minor units, that the destination wallet will receive once the deposit clears. The fiat-side fields (`memo_code`, `blindpay_bank_details`, `pix_code`, `clabe`) mirror whichever `payment_method` was set on the quote; only the relevant one is populated.

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

## What to display per method

| `payment_method` | What to show the payer | Field(s) in the response |
| --- | --- | --- |
| `ach` | `memo_code` plus BlindPay's bank details, so the deposit can be matched to this payin | `memo_code`, `blindpay_bank_details` |
| `wire` | Same as `ach` | `memo_code`, `blindpay_bank_details` |
| `rtp` | Same as `ach`, always | `memo_code`, `blindpay_bank_details` |
| `international_swift` | BlindPay's bank details, SWIFT-routed | `blindpay_bank_details` |
| `pix` | The Pix code as copyable text or a QR code | `pix_code` |
| `ted` | Same as `ach` | `memo_code`, `blindpay_bank_details` |
| `spei` | The CLABE number | `clabe` |
| `transfers` | The account number (CVU, CBU, or Alias, see `type`) | `tracking_transaction.transfers_instruction.account`, `tracking_transaction.transfers_instruction.type` |
| `pse` | The payment link | `tracking_transaction.pse_instruction.payment_link` |

**Note:**

If the customer has an approved [virtual account](../virtual-accounts/virtual-accounts.md), BlindPay displays their own dedicated account details instead, and `memo_code` is ignored even though the field is still returned. This applies to `ach` and `wire`.

`rtp` is the exception: no virtual account carries an RTP rail, so an `rtp` payin always uses the shared memo-code account, even when the customer has an approved virtual account for `ach`/`wire`. `international_swift` requires an approved virtual account and has no memo-code fallback at all.

**Advanced flavor (stablecoin and blockchain mechanics)**

## Stablecoin delivery

Once the fiat deposit is confirmed, BlindPay converts it and sends the equivalent stablecoins on-chain to the destination wallet you set on the payin quote (`blockchain_wallet_id` or `wallet_id`). Delivery happens automatically; there is no separate call to trigger it.

| Chain | Tokens |
| --- | --- |
| Ethereum, Polygon (EVM) | USDC, USDT |
| Base, Arbitrum (EVM) | USDC |
| Stellar | USDC |
| Solana | USDC, USDT |

BlindPay detects the destination's network from the wallet record itself, so the same `POST /payins/evm` call works for every chain above; you don't select a network explicitly on the payin.

**Note:**

Stellar mainnet deliveries originate from BlindPay's treasury wallet: `GCOSSQDM2SWMHRP7CDBOLL2V45NHCRLUWUCEHPPBA2ABCOOLPOLZKIHE`. This is the address that sends stablecoins to your blockchain wallet once the fiat payment is confirmed, and it is useful for reconciling incoming transactions on an explorer.

## Pull funding from a Plaid-connected account

With `payment_method: "ach_pull"`, instead of the payer sending a manual bank transfer, BlindPay pulls the funds directly from a bank account the customer connected through [Plaid](../payouts/bank-accounts.md#connect-with-plaid). Set `funding_bank_account_id` (a `ba_...` id) on the payin quote to a Plaid-connected account belonging to the same customer; it is required for `ach_pull`, and the quote rejects any other bank account with 400 `funding_account_not_plaid_connected`.

```bash [cURL]
curl https://api.blindpay.com/v1/instances/in_000000000000/payin-quotes \
  --request POST \
  --header 'Content-Type: application/json' \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --data '{
  "blockchain_wallet_id": "bw_000000000000",
  "currency_type": "sender",
  "cover_fees": true,
  "request_amount": 10000,
  "payment_method": "ach_pull",
  "token": "USDB",
  "funding_bank_account_id": "ba_000000000000"
}'
```

Creating the payin from that quote triggers the pull automatically; there is no `memo_code` or `blindpay_bank_details` for the payer to act on, and no manual transfer for BlindPay to wait for. If the pull cannot be initiated, the payin fails immediately with `PAYINS_FUNDING_PULL_FAILED` instead of sitting in `processing`.

BlindPay's ACH-pull provider charges a flat **$1.00** fee on every pull, deducted from the amount actually pulled: the payer's connected account is debited the quote's `sender_amount` (which already includes the fee), but the pull only moves `sender_amount` minus $1.00.

**Note:**

Use `payment_method: "ach"` without `funding_bank_account_id` to keep the default manual bank transfer flow described above.

## Monitoring window

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

Once created, a payin waits for the fiat to actually land before it converts anything. On development instances every payin auto-completes about 30 seconds after creation, regardless of method. On production, the wait depends on the rail:

**Advanced flavor (stablecoin and blockchain mechanics)**

On development instances every payin auto-completes about 30 seconds after creation, regardless of payment method. On production, BlindPay waits for the underlying fiat to actually land before converting and sending anything:

| `payment_method` | Currency | Typical arrival window |
| --- | --- | --- |
| `ach` | USD | up to 5 business days |
| `wire` | USD | up to 5 business days |
| `pix` | BRL | typically settles within minutes; BlindPay waits up to 30 minutes before marking a non-OTC `pix` payin `failed` (with a reconciliation check first). OTC `pix` payins wait until the [18:50 BRT cutoff](../kb/cut-off-times.md) instead |
| `ted` | BRL | up to 7 days (covers a Friday cut-off with next-week reconciliation) |
| `spei` | MXN | typically settles within minutes; BlindPay waits up to 30 minutes before marking the payin `failed` |
| `transfers` | ARS | typically settles within minutes; BlindPay waits up to 30 minutes before marking the payin `failed` |
| `pse` | COP | typically settles within minutes; BlindPay waits up to 30 minutes before marking the payin `failed`, since PSE's bank redirect and 2FA step can lag |

See [cut-off times](../kb/cut-off-times.md) for the full settlement-window reference.

**Note:**

If an OTC `pix` payin fails because the deposit never arrives, a flat $100.00 penalty is added to the quote's billing fee.

**Note:**

If the funding for a `transfers` payin arrives after it was already marked `failed`, and no stablecoins were sent yet, BlindPay automatically revives it back to `processing` and completes the transfer once it matches the sender's tax id and amount.

## Status lifecycle

| `status` | Meaning | Terminal? |
| --- | --- | --- |
| `processing` | Waiting for the deposit to arrive, or converting/sending stablecoins once it has | no |
| `on_hold` | Held for manual review (risk or compliance) before continuing | no |
| `completed` | Stablecoins delivered to the destination wallet | yes |
| `failed` | The payin did not go through | yes |
| `refunded` | The deposit was returned to the sender | yes |

Each `tracking_*` object on the payin (`tracking_transaction`, `tracking_payment`, `tracking_complete`, `tracking_partner_fee`) exposes a finer-grained `step` for the corresponding stage, useful for building a detailed status view:

| `step` | Meaning |
| --- | --- |
| `processing` | The stage is in progress |
| `on_hold` | Waiting to start, or held |
| `pending_review` | Held for manual review |
| `pending_refund_review` | A refund on this stage is pending manual review |
| `completed` | The stage finished |

Each `tracking_*` object also has its own `completed_at` timestamp, set once its `step` reaches `completed`.

`tracking_transaction` also carries its own `status` field, separate from `step`: `null` while the fiat leg is still pending, then `failed` or `completed` once it resolves. Don't confuse it with the payin's top-level `status`. It also carries `provider_name`, identifying the banking partner that processed the fiat leg for that rail.

**Note:**

If fees consume the entire amount and `receiver_amount` resolves to 0, the payin reaches `completed` immediately without an on-chain transfer, since a $0.00 transfer would be rejected by every chain. This is a rare edge case; virtual-account deposits are fee-capped so they always deliver at least $0.01, see [Fees on deposits](../virtual-accounts/virtual-accounts.md#fees-on-deposits).

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

**Warning:**

If a payin lands on-chain but the broadcast hash was replaced (for example during a gas spike), BlindPay automatically resolves the actual landed transaction. The `transaction_hash` you eventually see in `tracking_complete` may differ from the one you initially observed being broadcast; treat `status` as the source of truth, not a specific hash.

**Advanced flavor (stablecoin and blockchain mechanics)**

**Warning:**

If the on-chain delivery transaction gets replaced (for example during a gas spike), BlindPay automatically resolves the transaction that actually landed and re-broadcasts if needed. The `transaction_hash` you eventually see in `tracking_complete` may differ from the one you initially observed being broadcast; treat `status` as the source of truth, not a specific hash.

## Retrieve a payin

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID.

```bash [cURL]
curl https://api.blindpay.com/v1/instances/in_000000000000/payins/pi_000000000000 \
  --header 'Authorization: Bearer YOUR_API_KEY'
```

The full response carries more fields than the create-payin example above:

| Field | Notes |
| --- | --- |
| `token`, `network` | Destination stablecoin and chain, from the payin quote's wallet |
| `is_otc` | Whether the underlying quote was OTC |
| `currency` | The fiat currency of `sender_amount` |
| `commercial_quotation`, `blindpay_quotation` | Exchange rates locked at quote time |
| `total_fee_amount` | Sum of the fee fields |
| `first_name`, `last_name`, `legal_name`, `image_url`, `type` | The customer the payin belongs to |
| `pse_payment_link`, `pse_full_name`, `pse_tax_id`, `pse_document_type` | Set only for `pse` payins |
| `payer_rules` | The allowlist/instructions set on the quote |
| `manual_execution_status`, `manual_concluded_at`, `manual_concluded_by` | Set only for OTC payins routed to manual liquidity |

## List payins

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID.

```bash [cURL]
curl --url 'https://api.blindpay.com/v1/instances/in_000000000000/payins?status=processing&limit=10' \
  --header 'Authorization: Bearer YOUR_API_KEY'
```

| Query param | Notes |
| --- | --- |
| `customer_id` | Filter to one customer |
| `status` | One of the [status](#status-lifecycle) values |
| `customer_name` | Matches the customer's first + last name or legal name (partial match) |
| `bank_account_id` | Filter to payins whose funding pull used this bank account |
| `country` | ISO country code |
| `payment_method` | One of the [payment methods](#payment-methods) |
| `network` | Destination blockchain network |
| `token` | Destination stablecoin |
| `limit`, `starting_after`, `ending_before` | Cursor pagination |

Combine any of the filters above; they're ANDed together.

**Warning:**

If you pass none of `limit`, `starting_after`, or `ending_before`, the endpoint returns the full unpaginated array of matching payins instead of a `{ data, pagination }` envelope. Pass at least `limit` to get the paginated shape.

## Track a payin externally

`GET /e/payins/{id}` returns the same payin fields as [Retrieve a payin](#retrieve-a-payin), but requires no API key: anyone with the payin id can call it. Use it to power a public tracking page or a receipt for the payer, who never has an API key of their own.

```bash [cURL]
curl https://api.blindpay.com/v1/e/payins/pi_000000000000
```

The following fields are always removed from the response, even though they exist on the authenticated payin object: `id_doc_front_file`, `id_doc_back_file`, `selfie_file`, `proof_of_address_doc_file`, `source_of_funds_doc_file`, `incorporation_doc_file`, `proof_of_ownership_doc_file`, `transaction_document_file`, `transaction_document_id`, `aiprise_validation_key`, `kyc_warnings`, `signed_transaction`, `ip_address`. Every other field, including the receiver's name, tax id, and address, is returned unmodified since the tracking page needs them.

## Testing

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

On development instances every payin auto-completes about 30 seconds after creation. Force a specific outcome instead by setting the payin quote's `request_amount` to one of these sentinel values:

**Advanced flavor (stablecoin and blockchain mechanics)**

On development instances every payin auto-completes about 30 seconds after creation, using the sandbox `USDB` token. Force a specific outcome instead by setting the payin quote's `request_amount` to one of these sentinel values:

| `request_amount` (minor units) | Result |
| --- | --- |
| `66600` ($666.00) | `failed` |
| `77700` ($777.00) | `refunded` |
| any other amount | `completed` after the normal ~30 second delay |

**Note:**

On development instances, the payment instructions returned for `pix` (`pix_code`) and for the memo-code rails (`memo_code`) are the literal placeholder string `<development>`, not a real, scannable, or payable value. BlindPay never calls the payment provider in development. Test the real payer flow against a production instance.

## Webhooks

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

| Event | Fires when |
| --- | --- |
| `payin.new` | The payin is created |
| `payin.update` | The payin's status or tracking data changes |
| `payin.complete` | The payin reaches `completed` |

See [webhooks](../essentials/webhooks.md) for signature verification and payload details.

**Advanced flavor (stablecoin and blockchain mechanics)**

| Event | Fires when |
| --- | --- |
| `payin.new` | The payin is created |
| `payin.update` | The payin's status or tracking data changes |
| `payin.complete` | The payin reaches `completed`, meaning stablecoins were delivered to the destination wallet |

See [webhooks](../essentials/webhooks.md) for signature verification and full payload details.

**Note:**

On production instances with email notifications enabled, creating a `pix` payin also sends the receiver an automatic "Onramp Initiated" email with the payin id and the Pix code. Disable this from the dashboard if you handle payer notifications yourself.

## Related

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

- [Payin quotes](payin-quotes.md): lock the amount, method, and fee split before creating a payin
- [Virtual accounts](../virtual-accounts/virtual-accounts.md): give a customer their own dedicated account instead of a memo code
- [Payouts](../payouts/payouts.md): pay out from stablecoins to a bank account
- [Webhooks](../essentials/webhooks.md): event payloads and signature verification
- [Cut-off times](../kb/cut-off-times.md): settlement windows by payment method

**Advanced flavor (stablecoin and blockchain mechanics)**

- [Payin with managed wallet](payin-managed-wallet.md): the REST-only delivery path
- [Payin with blockchain wallet](payin-blockchain-wallet.md): deliver to a customer-controlled wallet
- [Payin quotes](payin-quotes.md): lock the amount, payment method, and destination wallet before creating a payin
- [Blockchain wallets](blockchain-wallets.md): add the external wallet that receives delivered stablecoins
- [Supported chains](../kb/supported-chains.md): full chain and token matrix
