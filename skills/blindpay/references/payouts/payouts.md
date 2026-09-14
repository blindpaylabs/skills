# Payouts

Execute a payout to a recipient's bank account and track it from processing to completed, failed, or refunded.

Source: https://blindpay.com/docs/payouts

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

Send covers paying fiat out to a recipient's bank account. As a bank-rails integrator you settle in stablecoins under the hood, but you only need to think in three steps: add a bank account, create a payout quote, and execute the payout.

## How it works

Every payout follows the same three-step pattern:

1. Add a bank account for the recipient (once per recipient)
2. Create a payout quote: locks the exchange rate and fees for 5 minutes
3. Execute the payout: moves funds from the funding source to the bank account

![BlindPay payout flow of funds, from funding source to recipient bank account](https://pub-4fabf5dd55154f19a0384b16f2b816d9.r2.dev/blindpay-payouts-flow-of-funds.png)

### Supported payout rails

| `type` | Country |
| --- | --- |
| `international_swift` | Global |
| `ach` | United States |
| `wire` | United States |
| `rtp` | United States |
| `pix` | Brazil |
| `spei_bitso` | Mexico |
| `ach_cop_bitso` | Colombia |
| `transfers_bitso` | Argentina |
| `sepa` | Europe (SEPA zone) |

**Note:**

You can add third-party bank accounts: a customer named "John" can have a payout sent to a bank account belonging to "Jack".

### Minimum amounts

| `type` | Minimum | Error when below |
| --- | --- | --- |
| `international_swift` | 100 USD requested amount | `swift_minimum_is_100_usd` |
| `sepa` | 11 USDC when quoting by sender amount, 10 EUR when quoting by receiver amount | `sepa_minimum_is_11_usdc`, `sepa_minimum_is_10_eur` |

The SWIFT minimum applies to the requested amount before fees, on both quote-based and offramp wallet payouts. Other rails have no minimum beyond the fee itself.

### Funding source

Every payout needs a funding source, the wallet the settlement stablecoins are pulled from.

- **BlindPay-managed wallet balance**: the simple path. A managed wallet is custodied by BlindPay on the customer's behalf, so there is no client-side signing. Pass its address as `sender_wallet_address` and BlindPay moves the funds directly.
- **External wallet**: if the funds live in a customer-controlled wallet instead, you must authorize the transfer on-chain first before calling the payout endpoint. See the stablecoin tutorials for the full mechanics: [EVM approve](payout-evm.md), [Stellar authorize-and-sign](payout-stellar.md), or [Solana delegation](payout-solana.md).

A payout executes the transfer locked in by a [payout quote](payout-quotes.md): stablecoins move out of the funding source and fiat lands in the recipient's [bank account](bank-accounts.md). As a bank-rails integrator, you call one endpoint and then track status; the settlement leg is plumbing.

**Note:**

The payout endpoint is the same regardless of funding source. What differs is whether you complete an on-chain authorization step first.

**Advanced flavor (stablecoin and blockchain mechanics)**

A payout executes the transfer locked in by a [payout quote](payout-quotes.md): stablecoins move out of a funding source and the equivalent fiat lands in a customer's [bank account](bank-accounts.md). This page is the payout reference: how funding paths differ, the response shape, statuses, testing, and webhooks. The step-by-step flows live in the per-path tutorials below.

## How it works

The pattern is always the same: create a [payout quote](payout-quotes.md), authorize the tokens for the chosen funding path, then create the payout before the quote expires (5 minutes by default). What changes is the authorization step:

| Funding source | Authorization | Tutorial |
| --- | --- | --- |
| [Managed wallet](../wallets/wallets.md) (`bl_...`) | None, BlindPay custodies the balance | [Payout with managed wallet](payout-managed-wallet.md) |
| Blockchain wallet on EVM (Ethereum, Base, Polygon, Arbitrum) | ERC-20 `approve` on the token contract, scoped to the quoted amount | [Payout with EVM](payout-evm.md) |
| Blockchain wallet on Stellar | Call the authorize endpoint for an unsigned XDR transaction, sign it, then create the payout | [Payout with Stellar](payout-stellar.md) |
| Blockchain wallet on Solana | Prepare a token delegation transaction, sign and submit it, then create the payout | [Payout with Solana](payout-solana.md) |

**Note:**

On development instances, use the test networks (`base_sepolia`, `stellar_testnet`, `solana_devnet`) and the `USDB` test token; fund test wallets with a [payin](../payins/payins.md), which auto-completes on development instances. Production instances use the matching mainnet and `USDC` or `USDT`.

## Prerequisites

**Before you start:**

1. Create an account at https://app.blindpay.com/sign-up
2. Create a development instance (see essentials/instances.md)
3. Create your API key (see essentials/api-keys.md)

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

You also need a [customer](../getting-started/overview.md) who has completed KYC, an [approved bank account](bank-accounts.md), and an unexpired [payout quote](payout-quotes.md).

## Execute a payout

Check the required fields in the [BlindPay API Docs](https://api.blindpay.com/reference#tag/payouts/POST/v1/instances/{instance_id}/payouts/evm){target="_blank"}.

**Advanced flavor (stablecoin and blockchain mechanics)**

Once the funding path is authorized, one call creates the payout. EVM, Solana, and managed-wallet payouts all use `/payouts/evm`; only Stellar has its own endpoint (`/payouts/stellar`, which also takes the signed transaction).

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID, `re_000000000000` with your customer ID, `ba_000000000000` with your bank account ID.

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/payouts/evm \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
  "quote_id": "qu_000000000000",
  "sender_wallet_address": "YOUR_WALLET_ADDRESS"
}'
```

```js [index.js]
const response = await fetch(
  'https://api.blindpay.com/v1/instances/in_000000000000/payouts/evm',
  {
    method: 'POST',
    headers: {
      'Authorization': 'Bearer YOUR_API_KEY',
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      quote_id: 'qu_000000000000',
      sender_wallet_address: 'YOUR_WALLET_ADDRESS',
    }),
  }
)

const payout = await response.json()
```

**Warning:**

Payout creation is split by destination network, each with its own endpoint and request shape: `/payouts/evm`, `/payouts/stellar`, and `/payouts/solana`. Which one you call depends on the recipient's network, not on the funding source; see [Payout with Stellar](payout-stellar.md) and [Payout with Solana](payout-solana.md) for those request shapes.

`quote_id` can only back one payout. A second call with the same quote returns an error; create a new quote instead.

**Advanced flavor (stablecoin and blockchain mechanics)**

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/payouts/evm \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
  "quote_id": "qu_000000000000",
  "sender_wallet_address": "YOUR_WALLET_ADDRESS"
}'
```

`quote_id` can only back one payout; a second call with the same quote fails, create a new quote instead.

### Response

```json
{
  "id": "po_000000000000",
  "status": "processing",
  "sender_wallet_address": "0x...",
  "customer_id": "re_000000000000",
  "bank_account_id": "ba_000000000000",
  "offramp_wallet_id": null,
  "billing_fee_amount": null,
  "partner_fee": 0,
  "tracking_complete": { "step": "processing" },
  "tracking_payment": { "step": "processing" },
  "tracking_transaction": { "step": "processing" },
  "tracking_partner_fee": { "step": "on_hold" },
  "tracking_liquidity": { "step": "processing" }
}
```

| Field | Type | Notes |
| --- | --- | --- |
| `id` | string | The payout ID (`po_...`). |
| `status` | string | See status lifecycle below. |
| `sender_wallet_address` | string | The funding source address the crypto was pulled from. |
| `customer_id` | string | The customer this payout belongs to (`re_...`). |
| `bank_account_id` | string | The recipient bank account (`ba_...`). |
| `partner_fee` | number | Nonzero only if the quote referenced a `partner_fee_id`. See [partner fees](../essentials/partner-fees.md). |
| `tracking_complete`, `tracking_payment`, `tracking_transaction`, `tracking_partner_fee`, `tracking_liquidity` | object | Sub-status objects with a `step` of `processing`, `on_hold`, `pending_review`, or `completed`. Poll these or rely on webhooks for progress detail beyond the top-level `status`. |

**Advanced flavor (stablecoin and blockchain mechanics)**

## cover_fees

`cover_fees` on the payout quote decides who absorbs the fee, and it's locked in before you authorize the tokens:

| `cover_fees` | Who pays | Effect |
| --- | --- | --- |
| `false` | Customer | Fees are deducted from the fiat the bank account receives (the common case) |
| `true` | Sender | Fees are added on top, sent as extra stablecoin, so the recipient gets the full quoted amount |

See [payout quotes](payout-quotes.md) for the full request and response fields.

## Status lifecycle

| Status | Meaning |
| --- | --- |
| `processing` | Default on creation. The stablecoins are being pulled from the funding source and the fiat transfer is in flight. |
| `on_hold` | Held for review. All SWIFT payouts start here. ACH, wire, and RTP payouts also pass through `on_hold` as a standard step, for compliance review after the crypto is collected. |
| `completed` | The fiat landed in the recipient's bank account. Terminal. |
| `failed` | The payout did not complete (for example, a review timeout or a rejected compliance check). Terminal. |
| `refunded` | The stablecoins were returned to the funding source instead of being converted to fiat. Terminal. |

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

**Note:**

SWIFT payouts always start `on_hold` and stay there until compliance documents are submitted and approved. See [Payout quotes](payout-quotes.md#swift-compliance-documents) for the document submission flow.

**Warning:**

A payout that reaches `failed` or sits past review does not automatically refund the customer. If you see a payout stuck in review, contact support@blindpay.com rather than assuming it will resolve on its own.

**Advanced flavor (stablecoin and blockchain mechanics)**

**Warning:**

A payout that reaches `failed` or sits in review does not automatically refund the sender. If a payout looks stuck, contact support@blindpay.com rather than assuming it resolves on its own.

## Testing

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

Development instances complete payouts automatically. Force a `failed` or `refunded` outcome by setting the payout quote's `request_amount` to one of these sentinel values before executing the payout:

| `request_amount` | Result |
| --- | --- |
| `66600` ($666.00) | `failed` |
| `77700` ($777.00) | `refunded` |
| any other amount | `completed` |

**Advanced flavor (stablecoin and blockchain mechanics)**

Development instances complete payouts automatically. Force a `failed` or `refunded` outcome by setting the payout quote's `request_amount` to one of these sentinel values before authorizing and executing the payout:

| `request_amount` | Result |
| --- | --- |
| `66600` ($666.00) | `failed` |
| `77700` ($777.00) | `refunded` (EVM payouts also fire a real on-chain refund transaction) |
| any other amount | `completed` |

## Webhooks

**Note:**

Configure a webhook endpoint once per instance. See [Webhooks](../essentials/webhooks.md) for setup and signature verification.

| Event | Fires when |
| --- | --- |
| `payout.new` | A payout is created. |
| `payout.update` | A payout changes status (for example, `processing` to `on_hold`). |
| `payout.complete` | A payout reaches `completed`, `failed`, or `refunded`. |
| `payout.partnerFee` | A payout completes and a partner fee is owed. See [Partner fees](../essentials/partner-fees.md). |

A payout can also execute a registered bill instead of a bank account: its `payable_id` is set, the bank fields are null, and the [payable](../payables/payables.md) emits its own `payable.*` events alongside these. Correlate the two through `payable_id`/`payout_id` and never count both completes as two payments.

## Related

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

- [Payout quotes](payout-quotes.md): lock the exchange rate and fee split before executing
- [Payables](../payables/payables.md): register a bill (boleto, arrecadação, PIX code or US invoice) and pay it through this same flow
- [Bank accounts](bank-accounts.md): add and manage recipient bank accounts
- [Payout with EVM](payout-evm.md), [Stellar](payout-stellar.md), and [Solana](payout-solana.md): on-chain authorization mechanics for external wallets
- [Payins](../payins/payins.md): accept fiat, and fund test wallets on development instances
- [Webhooks](../essentials/webhooks.md): full event catalogue and signature verification

**Advanced flavor (stablecoin and blockchain mechanics)**

- [Payout with managed wallet](payout-managed-wallet.md): the REST-only funding path
- [Payout with EVM](payout-evm.md): ERC-20 approve from an external wallet
- [Payout with Stellar](payout-stellar.md): authorize, sign the XDR, create
- [Payout with Solana](payout-solana.md): delegate the tokens, then create
- [Payout quotes](payout-quotes.md): lock the exchange rate, fees, and the on-chain approval payload
- [Payin with managed wallet](../payins/payin-managed-wallet.md): fund test wallets on development instances
- [Supported chains](../kb/supported-chains.md): the full chain and token matrix
