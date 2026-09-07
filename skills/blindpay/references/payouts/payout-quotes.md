# Payout quotes

Lock the exchange rate and fee split before executing a payout, and see the exact fee breakdown and recipient amount in the response.

Source: https://blindpay.com/docs/payout-quotes

A payout quote locks the exchange rate and fee for a payout before you execute it. A payout can only execute against a valid, unexpired quote.

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

It tells you exactly how much leaves the funding source and how much the recipient's bank account receives.

**Advanced flavor (stablecoin and blockchain mechanics)**

For a stablecoin-funded payout, the quote also returns the on-chain payload you need to authorize the token transfer: the ERC-20 contract address, ABI, and the exact decimal-adjusted amount for EVM chains.

## How it works

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

A payout quote is created against a specific [bank account](bank-accounts.md) and expires 5 minutes after creation. The response includes the exact fee breakdown and the final amount the recipient receives, so you can show the customer a firm number before committing to the payout.

For the fiat lens, treat the stablecoin leg as settlement plumbing: you pass a `network` and `token` to price the quote, but the funding source itself (a managed wallet balance in the common case) is covered on [Payouts](payouts.md).

**Advanced flavor (stablecoin and blockchain mechanics)**

A payout quote is created against a specific [bank account](bank-accounts.md) and a `network` + `token` pair for the funding leg. It expires 5 minutes after creation. The response includes the fee breakdown, the final fiat amount the recipient receives, and (for EVM networks) the `contract` payload used to authorize the token pull.

### network and token

`network` and `token` select the chain and stablecoin the funding source sends from. Availability depends on your instance type:

| Instance | Networks | Tokens |
| --- | --- | --- |
| Development | `sepolia`, `base_sepolia`, `arbitrum_sepolia`, `polygon_amoy`, `stellar_testnet`, `solana_devnet` | `USDB` only |
| Production | `ethereum`, `base`, `polygon`, `arbitrum`, `stellar`, `solana`, `tron` (beta) | `USDC` or `USDT` |

Token and chain support also differ by token:

| Token | Supported networks |
| --- | --- |
| `USDC` | `polygon`, `base`, `arbitrum`, `ethereum`, `stellar`, `solana` (and their dev equivalents) |
| `USDT` | `polygon`, `ethereum`, `solana`, `tron` (and the dev equivalents, except `tron`) |
| `USDB` | development testnets only: `polygon_amoy`, `base_sepolia`, `arbitrum_sepolia`, `sepolia`, `stellar_testnet`, `solana_devnet` |

`tron` is a production-only network in beta: it has no development testnet, so it can't be exercised on a development instance. See [Supported chains](../kb/supported-chains.md) for details.

Passing a network and token that are not compatible returns a 400 with a message naming the unsupported pair. See [Supported chains](../kb/supported-chains.md) for the full matrix.

### currency_type

`currency_type` tells the API which side of the payout `request_amount` is denominated in. On a payout quote, this is the opposite direction from a payin quote:

| `currency_type` | `request_amount` is denominated in |
| --- | --- |
| `sender` | The stablecoin the funding source sends (the token leg) |
| `receiver` | The fiat currency the bank account receives |

**Warning:**

Payin quotes use the same field name with the opposite meaning: on a payin quote, `sender` means the fiat the payer sends. Always check which quote type you're building against.

### cover_fees

Fees can be paid by either party:

| Payer | Fee basis | API setting | Dashboard option |
| --- | --- | --- | --- |
| Customer | Deducted from the fiat amount the recipient receives | `cover_fees: false` | Keep "Cover all payout fees" off |
| Sender | Added on top of the stablecoin amount sent, so the recipient receives the full amount | `cover_fees: true` | Enable "Cover all payout fees" |

Customer-paid fees are the most common case. Sender-paid fees are typical for payroll, where the company wants the recipient to receive an exact amount.

![Payout quote with cover_fees true: the sender pays the fees](https://pub-4fabf5dd55154f19a0384b16f2b816d9.r2.dev/sender_fees.jpg)

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

`request_amount` is an integer in minor units; it does not accept floats. To send `$100.00`, pass `10000`.

**Advanced flavor (stablecoin and blockchain mechanics)**

`request_amount` is an integer in minor units; it does not accept floats. To send `100 USDC`, pass `10000`.

### SWIFT compliance documents

SWIFT payouts need compliance documents, but they are collected **after** the payout is created, not at quote creation time. Once you create the payout it is placed `on_hold` until the required documents are submitted and approved.

**Note:**

Documents are only required when the recipient relationship is not `first_party` (sending to a third party). Sending to your own SWIFT account never requires documents.

### Optional documents on other rails

Payouts on any other rail (wire, ACH, RTP, Pix and so on) never wait for documents, but you can still attach one for our compliance team. Call the same document submission endpoint while the payout is `processing` or `completed`. The document is stored on the payout and shown to compliance; the payout `status` and `tracking_documents` are not changed, the `description` field is ignored, and submitting again replaces the previous document. Payouts that are `failed`, `refunded` or `on_hold` return `payout_not_processing_or_completed`.

## Prerequisites

**Before you start:**

1. Create an account at https://app.blindpay.com/sign-up
2. Create a development instance (see essentials/instances.md)
3. Create your API key (see essentials/api-keys.md)

You also need a [customer](../getting-started/overview.md) who has completed KYC and a [bank account](bank-accounts.md) with `status: "approved"`.

## Create a payout quote

Check the required fields in the [BlindPay API Docs](https://api.blindpay.com/reference#tag/quotes/POST/v1/instances/{instance_id}/quotes){target="_blank"}.

The quote accepts exactly one destination: `bank_account_id` (shown below) or `payable_id` to pay a registered bill. See [Payables](../payables/payables.md#how-to-pay-one) for that variant, where the amount comes from the bill itself.

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID, `re_000000000000` with your customer ID, `ba_000000000000` with your bank account ID.

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/quotes \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
  "bank_account_id": "ba_000000000000",
  "currency_type": "sender",
  "cover_fees": false,
  "request_amount": 10000,
  "network": "sepolia",
  "token": "USDB"
}'
```

```js [index.js]
const response = await fetch(
  'https://api.blindpay.com/v1/instances/in_000000000000/quotes',
  {
    method: 'POST',
    headers: {
      'Authorization': 'Bearer YOUR_API_KEY',
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      bank_account_id: 'ba_000000000000',
      currency_type: 'sender',
      cover_fees: false,
      request_amount: 10000,
      network: 'sepolia',
      token: 'USDB',
    }),
  }
)

const quote = await response.json()
```

**Advanced flavor (stablecoin and blockchain mechanics)**

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/quotes \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
  "bank_account_id": "ba_000000000000",
  "currency_type": "sender",
  "cover_fees": false,
  "request_amount": 10000,
  "network": "sepolia",
  "token": "USDC"
}'
```

```js [index.js]
const response = await fetch(
  'https://api.blindpay.com/v1/instances/in_000000000000/quotes',
  {
    method: 'POST',
    headers: {
      'Authorization': 'Bearer YOUR_API_KEY',
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      bank_account_id: 'ba_000000000000',
      currency_type: 'sender',
      cover_fees: false,
      request_amount: 10000,
      network: 'sepolia',
      token: 'USDC',
    }),
  }
)

const quote = await response.json()
```

### Response

**Advanced flavor (stablecoin and blockchain mechanics)**

For EVM networks, the response includes a `contract` object: the payload you pass to your Ethereum library to call `approve` on the token contract, authorizing BlindPay to pull the quoted amount.

```json
{
  "id": "qu_000000000000",
  "expires_at": 1712958191000,
  "commercial_quotation": 1,
  "blindpay_quotation": 0.998,
  "sender_amount": 10000,
  "receiver_amount": 9980,
  "partner_fee_amount": 0,
  "flat_fee": 20,
  "billing_fee_amount": null,
  "contract": {
    "address": "0x...",
    "abi": [],
    "functionName": "approve",
    "blindpayContractAddress": "0x...",
    "amount": "100000000",
    "network": {
      "name": "sepolia",
      "chainId": 11155111
    }
  }
}
```

| Field | Type | Notes |
| --- | --- | --- |
| `id` | string | The quote ID (`qu_...`). References the same prefix as payin and transfer quotes. |
| `expires_at` | number | Epoch **milliseconds**. |
| `commercial_quotation` | number | The raw market exchange rate. |
| `blindpay_quotation` | number | The rate net of BlindPay's fee. |
| `sender_amount` | number | The stablecoin amount the funding source sends, in minor units. |
| `receiver_amount` | number | The fiat amount the bank account receives, in minor units. |
| `partner_fee_amount` | number | Nonzero only if `partner_fee_id` was passed. See [partner fees](../essentials/partner-fees.md). |
| `flat_fee` | number | The flat-fee component. |
| `billing_fee_amount` | number, nullable | Only nonzero on instances with billing charges enabled. |

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

| `contract` | object, nullable | The on-chain ERC-20 `approve` payload for the chosen token and network. Only relevant when funding from a self-custodied wallet; see [Payout with EVM](payout-evm.md). |

**Note:**

Save the quote ID (`qu_...`). It expires in 5 minutes; if it expires before you execute the payout, create a new one.

**Advanced flavor (stablecoin and blockchain mechanics)**

| `contract` | object, nullable | Present for EVM networks. The on-chain `approve` payload; see below. |

`contract` fields:

| Field | Type | Notes |
| --- | --- | --- |
| `contract.address` | string | The ERC-20 token contract address for the chosen token and network. |
| `contract.abi` | array | The ERC-20 ABI, ready to pass to your Ethereum library. |
| `contract.functionName` | string | Always `approve`. |
| `contract.blindpayContractAddress` | string | The address to approve as spender: BlindPay's receiving address on that network. |
| `contract.amount` | string | The decimal-adjusted amount to approve, as a string. |
| `contract.network` | object | `{ name, chainId }` for the quoted network. |

Stellar and Solana don't use the allowance pattern, so `contract` is not meaningful there: Stellar payouts sign an XDR transaction directly, and Solana payouts sign a token delegation. See [Payout with Stellar](payout-stellar.md) and [Payout with Solana](payout-solana.md) for the full authorization flows.

**Note:**

Save the quote ID (`qu_...`). It expires in 5 minutes; if it expires before you execute the payout, create a new one.

## Related

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

- [Payouts](payouts.md): execute the payout against this quote
- [Bank accounts](bank-accounts.md): add and manage recipient bank accounts
- [Partner fees](../essentials/partner-fees.md): pass a `partner_fee_id` to earn a cut of the payout
- [Cut-off times](../kb/cut-off-times.md): settlement windows by payment rail

**Advanced flavor (stablecoin and blockchain mechanics)**

- [Payouts](payouts.md): the payout reference, status lifecycle, and links to every funding-path tutorial
- [Bank accounts](bank-accounts.md): add and manage recipient bank accounts
- [Supported chains](../kb/supported-chains.md): the full chain and token matrix
- [Partner fees](../essentials/partner-fees.md): pass a `partner_fee_id` to earn a cut of the payout
