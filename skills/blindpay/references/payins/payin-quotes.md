# Payin quotes

Lock the amount, fee split, and destination for a payin before creating it.

Source: https://blindpay.com/docs/payin-quotes

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

A payin quote locks in the numbers for a bank deposit before you commit to it: how much fiat the sender pays, how much the customer receives, the fee split, and the destination that settles the funds. You create a payin quote first, then create the [payin](payins.md) itself by referencing the quote's id. The quote is what actually enforces amount limits, currency rules, and payer requirements, so most of the validation work happens here rather than at payin creation.

## How it works

A payin quote is a single call: pass the amount, the payment method, the fee setting, and the destination. BlindPay returns the locked-in fiat and stablecoin amounts plus the payment instructions for that method. The quote expires in 5 minutes, so create the payin shortly after.

**Advanced flavor (stablecoin and blockchain mechanics)**

A payin quote locks in the numbers for an on-ramp before you commit to it: how much fiat the sender pays, how much stablecoin the destination wallet receives, the fee split, and the wallet the funds settle to. You create a payin quote first, then create the [payin](payins.md) itself by referencing the quote's id. The quote is what enforces amount limits, currency rules, and payer requirements, so most of the validation happens here rather than at payin creation.

```
payin quote -> payin (create within 5 minutes)
```

## Destination

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

The destination is a stablecoin delivery target, not a bank account. Pass exactly one of:

**Advanced flavor (stablecoin and blockchain mechanics)**

The destination is a stablecoin wallet, never a bank account. Pass exactly one of:

| Field | Points to | Prefix |
| --- | --- | --- |
| `blockchain_wallet_id` | An external blockchain wallet the customer controls | `bw_` |
| `wallet_id` | A BlindPay-managed wallet | `bl_` |

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

You cannot pass both, and you cannot pass neither. The stablecoin mechanics behind this destination are covered in [payins](payins.md); as a bank-rails integration you can treat it as an implementation detail.

**Advanced flavor (stablecoin and blockchain mechanics)**

You cannot pass both, and you cannot pass neither: BlindPay rejects the request if either rule is violated. BlindPay reads the network directly off the destination wallet, so you never pass a network on the quote itself. See [wallets](../wallets/wallets.md) and [blockchain wallets](blockchain-wallets.md) for how to register each type.

## Token and delivery network

`token` is the stablecoin the destination wallet receives. The network is implied by the wallet you pass as the destination, and only certain token and network combinations have a deployed contract:

| Chain | Tokens |
| --- | --- |
| Ethereum, Base, Polygon, Arbitrum (EVM) | USDC, USDT (USDT only on Polygon and Ethereum) |
| Stellar | USDC |
| Solana | USDC, USDT |
| Tron | USDT only |

**Note:**

Development instances only support the `USDB` test stablecoin, delivered on testnets (`sepolia`, `base_sepolia`, `arbitrum_sepolia`, `polygon_amoy`, `stellar_testnet`, `solana_devnet`). Production instances only support `USDC` or `USDT`, delivered on mainnets. The examples below use `USDB` since they target a development instance.

## currency_type

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

`currency_type` tells BlindPay which side `request_amount` is denominated in. On a payin quote, this is the opposite convention from a payout quote, so read it carefully:

| `currency_type` | `request_amount` is denominated in |
| --- | --- |
| `sender` | The fiat currency the sender sends (determined by `payment_method`) |
| `receiver` | The stablecoin the customer receives |

**Advanced flavor (stablecoin and blockchain mechanics)**

`currency_type` tells BlindPay which side `request_amount` is denominated in. On a payin quote, `sender` means fiat, which is the opposite convention from a payout quote (where `sender` means stablecoin), so read it carefully:

| `currency_type` | `request_amount` is denominated in |
| --- | --- |
| `sender` | The fiat currency the sender sends (determined by `payment_method`) |
| `receiver` | The stablecoin the destination wallet receives |

## cover_fees

Fees can be paid by either party:

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

| `cover_fees` | Who pays | Fee is deducted from |
| --- | --- | --- |
| `false` | The customer | The stablecoin amount the customer receives (most common) |
| `true` | The sender | Added on top of the fiat amount the sender sends |

**Advanced flavor (stablecoin and blockchain mechanics)**

| `cover_fees` | Who pays | Fee is deducted from |
| --- | --- | --- |
| `false` | The customer | The stablecoin amount the destination wallet receives (most common) |
| `true` | The sender | Added on top of the fiat amount the sender sends |

![Payin quote with cover_fees false: the customer pays the fees](https://pub-4fabf5dd55154f19a0384b16f2b816d9.r2.dev/blindpay-payin-quote-cover-fees-false-min.jpg)

![Payin quote with cover_fees true: the sender pays the fees](https://pub-4fabf5dd55154f19a0384b16f2b816d9.r2.dev/blindpay-payin-quote-cover-fees-true-min.jpg)

## request_amount

`request_amount` is an integer in minor units and does not accept floats. To send `$123.45`, pass `12345`.

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

Minimum and maximum amounts vary by currency, and the quote enforces them for you: if `request_amount` is outside the allowed range for that currency, the quote request fails with a dynamic error naming the min and max. Most currencies allow amounts as low as roughly $10 equivalent, but some (for example COP) require a much higher minimum in raw minor units because of the currency's smaller nominal value. Don't hardcode a single minimum across currencies; read the error if you hit the floor.

**Note:**

The examples below use the `USDB` test stablecoin, only available on development instances. In production use `USDC` or `USDT`.

**Advanced flavor (stablecoin and blockchain mechanics)**

Minimum and maximum amounts vary by currency, and the quote enforces them for you: if `request_amount` falls outside the allowed range for that currency, the request fails with a dynamic error naming the min and max. Most currencies allow amounts as low as roughly $10 equivalent, but some (for example COP) require a much higher minimum in raw minor units because of the currency's smaller nominal value. Don't hardcode a single minimum across currencies; read the error if you hit the floor.

### Whole units for MXN, COP, and ARS

When `currency_type` is `sender` and the sender currency is MXN, COP, or ARS, `request_amount` must be a multiple of 100 (a whole peso, no centavos): `10000` is valid, `10050` is rejected with `request_amount_must_be_a_whole_currency_unit`. These rails settle in whole units at BlindPay's banking partner, so a fractional amount would be truncated on arrival and the deposit could come back as an invalid payment amount.

This also means `sender_amount` on the response is always a whole unit for these three currencies, even when you request the receiver side (`currency_type: "receiver"`): BlindPay rounds the computed sender amount down to the nearest 100 minor units before returning it.

## Prerequisites

**Before you start:**

1. Create an account at https://app.blindpay.com/sign-up
2. Create a development instance (see essentials/instances.md)
3. Create your API key (see essentials/api-keys.md)

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

You also need a customer with a blockchain wallet or a virtual account.

**Advanced flavor (stablecoin and blockchain mechanics)**

You also need a customer with a [blockchain wallet](blockchain-wallets.md) or a [managed wallet](../wallets/wallets.md).

## Create a payin quote

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID, `re_000000000000` with your customer ID.

```bash [🇺🇸 ACH]
curl https://api.blindpay.com/v1/instances/in_000000000000/payin-quotes \
  --request POST \
  --header 'Content-Type: application/json' \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --data '{
  "blockchain_wallet_id": "bw_000000000000",
  "currency_type": "sender",
  "cover_fees": true,
  "request_amount": 10000,
  "payment_method": "ach",
  "token": "USDB"
}'
```

```bash [🇺🇸 Wire]
curl https://api.blindpay.com/v1/instances/in_000000000000/payin-quotes \
  --request POST \
  --header 'Content-Type: application/json' \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --data '{
  "blockchain_wallet_id": "bw_000000000000",
  "currency_type": "sender",
  "cover_fees": true,
  "request_amount": 10000,
  "payment_method": "wire",
  "token": "USDB"
}'
```

```bash [🇧🇷 Pix]
curl https://api.blindpay.com/v1/instances/in_000000000000/payin-quotes \
  --request POST \
  --header 'Content-Type: application/json' \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --data '{
  "blockchain_wallet_id": "bw_000000000000",
  "currency_type": "sender",
  "cover_fees": false,
  "request_amount": 10000,
  "payment_method": "pix",
  "token": "USDB",
  "payer_rules": {
    "pix_allowed_tax_ids": [
      "14747677786"
    ]
  }
}'
```

```bash [🇲🇽 SPEI]
curl https://api.blindpay.com/v1/instances/in_000000000000/payin-quotes \
  --request POST \
  --header 'Content-Type: application/json' \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --data '{
  "blockchain_wallet_id": "bw_000000000000",
  "currency_type": "sender",
  "cover_fees": false,
  "request_amount": 100000,
  "payment_method": "spei",
  "token": "USDB"
}'
```

```bash [🇦🇷 Transfers]
curl https://api.blindpay.com/v1/instances/in_000000000000/payin-quotes \
  --request POST \
  --header 'Content-Type: application/json' \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --data '{
  "blockchain_wallet_id": "bw_000000000000",
  "currency_type": "sender",
  "cover_fees": false,
  "request_amount": 2000000,
  "payment_method": "transfers",
  "token": "USDB",
  "payer_rules": {
    "transfers_allowed_tax_id": "30-27383762-7"
  }
}'
```

```bash [🇨🇴 PSE]
curl https://api.blindpay.com/v1/instances/in_000000000000/payin-quotes \
  --request POST \
  --header 'Content-Type: application/json' \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --data '{
  "blockchain_wallet_id": "bw_000000000000",
  "currency_type": "sender",
  "cover_fees": false,
  "request_amount": 20000000,
  "payment_method": "pse",
  "token": "USDB",
  "payer_rules": {
    "pse_full_name": "<Replace with payer full name>",
    "pse_document_type": "NIT",
    "pse_document_number": "<Replace with payer document number>",
    "pse_email": "<Replace with payer email>",
    "pse_phone": "<Replace with payer phone number>",
    "pse_bank_code": "<Replace with payer bank code>"
  }
}'
```

**Advanced flavor (stablecoin and blockchain mechanics)**

To target a managed wallet instead of a blockchain wallet, replace `blockchain_wallet_id` with `wallet_id` (`bl_000000000000`) in any of the payloads above.

### payer_rules

Some payment methods require payer identity fields so BlindPay can match and screen the incoming deposit:

| `payment_method` | `payer_rules` field | Notes |
| --- | --- | --- |
| `pix` | `pix_allowed_tax_ids` | Array of CPF/CNPJ tax ids allowed to send this Pix |
| `transfers` | `transfers_allowed_tax_id` | CUIT/CUIL tax id, required for `transfers` |
| `pse` | `pse_full_name`, `pse_document_type`, `pse_document_number`, `pse_email`, `pse_phone`, `pse_bank_code` | Full payer details, required for `pse` |

### Field formats

Some `payer_rules` fields are validated against a specific format, not just checked for presence:

| Field | Format |
| --- | --- |
| `pix_allowed_tax_ids` | CPF (11 digits) or CNPJ (14 digits), formatted or not. Validated against the Receita Federal check digits, not just digit count, so an invalid tax id is rejected at quote time instead of failing later at the bank |
| `transfers_allowed_tax_id` | CUIT/CUIL, for example `20-12345678-3` |
| `pse_document_type` | `CC` or `NIT` |
| `pse_phone` | `+573` followed by 9 digits, for example `+573001234567` |
| `pse_full_name` | Up to 50 characters |

## ach_pull and funding_bank_account_id

`payment_method: "ach_pull"` has BlindPay pull the funds from a bank account the customer already connected through [Plaid](../payouts/bank-accounts.md#connect-with-plaid), instead of the payer sending a manual bank transfer. It is priced like `ach` and settles in USD.

`funding_bank_account_id` (a `ba_...` id) is **required** on an `ach_pull` quote; omitting it fails with 400 `funding_bank_account_id_required`. The account must belong to the same customer and be Plaid-connected (`plaid_connected_at` set); otherwise the quote is rejected with 400 `funding_account_not_plaid_connected`. Passing `funding_bank_account_id` with any other `payment_method` fails with 400 `funding_account_invalid`. See [Payins](payins.md#pull-funding-from-a-plaid-connected-account) for the full pull flow.

Pulling this way also requires the `plaid` subscription feature on the instance itself. On an instance without it, an `ach_pull` quote fails with `BANK_ACCOUNTS_PLAID_NOT_ENABLED` before BlindPay even looks up the account. Contact BlindPay to enable Plaid for your instance.

Pulling through Plaid adds a flat **$1.00** fee on top of `sender_amount`, since BlindPay's ACH-pull provider charges it back to the payer. The `sender_amount` this endpoint returns already includes that fee, so it's the full amount to show the payer; the pull itself moves `sender_amount` minus the $1.00 fee.

```bash [🇺🇸 ACH pull]
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

## Response

| Field | Type | Notes |
| --- | --- | --- |
| `id` | string (`qu_`) | Pass this as `payin_quote_id` when creating the payin |
| `expires_at` | number | Epoch milliseconds. The quote is valid for 5 minutes |
| `sender_amount` | number | Fiat amount, in minor units, the sender must pay |
| `receiver_amount` | number | Stablecoin amount the destination wallet receives |
| `commercial_quotation` | number | Raw market exchange rate |
| `blindpay_quotation` | number | Exchange rate including BlindPay's fee |
| `flat_fee` | number | Flat-fee component of the quote |
| `partner_fee_amount` | number | Nonzero only when `partner_fee_id` is set |
| `billing_fee_amount` | number | Nonzero only when this quote's fee is invoiced at month end instead of charged now. See [Billing](../essentials/billing.md) |
| `is_otc` | boolean | Whether this is an OTC quote |

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

Show the payer whichever field is relevant to their payment method once you create the payin: `memo_code` and `blindpay_bank_details` for `ach`/`wire`, `pix_code` for `pix`, a CLABE for `spei`, a CBU for `transfers`, or a payment link for `pse`. Those fields live on the [payin](payins.md) response, not the quote.

**Advanced flavor (stablecoin and blockchain mechanics)**

The stablecoins themselves aren't sent yet at this point: creating the quote only locks the numbers. The [payin](payins.md) you create from this quote is what triggers the fiat collection and the on-chain delivery to the destination wallet.

### What the payin shows the payer

The payin quote's `id` doesn't carry payer-facing instructions; those appear once you create the [payin](payins.md) from the quote. Depending on `payment_method`, the payin response returns:

| `payment_method` | Field | Notes |
| --- | --- | --- |
| `ach`, `wire` | `memo_code` and `blindpay_bank_details` | Include the memo code with the transfer so BlindPay can match it. Ignored when the customer has an approved virtual account, since the payer sends to their own dedicated account instead |
| `pix` | `pix_code` | The Pix code (copia e cola) for the payer to complete the transfer |

See [payins](payins.md) for the full response reference and field descriptions.

## Expiry

A payin quote expires **5 minutes** after creation. Create the payin before then; an expired quote is rejected when you try to use it. Generate a new quote if the window has passed.

## Partner fees

Pass `partner_fee_id` (prefix `pf_`) to attribute a payin to a partner fee configured in the dashboard. The fee is snapshotted at quote time and reflected in `partner_fee_amount` on the response. See [partner fees](../essentials/partner-fees.md) for how the fee is calculated and collected.

## Preview an FX rate without creating a quote

`POST /instances/{instance_id}/payin-quotes/fx` returns an indicative rate and amount without creating a payin quote. Use it to show a live price as the payer types, before committing to the priced, time-limited quote that `POST /payin-quotes` returns.

Unlike `/payin-quotes`, this endpoint does not persist anything and does not reserve pricing: the numbers it returns aren't guaranteed to match the quote you get when you actually create one a moment later.

```bash [cURL]
curl https://api.blindpay.com/v1/instances/in_000000000000/payin-quotes/fx \
  --request POST \
  --header 'Content-Type: application/json' \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --data '{
  "from": "BRL",
  "to": "USDC",
  "request_amount": 10000,
  "currency_type": "sender"
}'
```

| Field | Notes |
| --- | --- |
| `from` | The fiat currency the sender pays in, for example `BRL` |
| `to` | The stablecoin the destination would receive: `USDC`, `USDT`, or `USDB` |
| `request_amount` | Minimum $5.00 (500 minor units). The same [whole-unit rule](#whole-units-for-mxn-cop-and-ars) as `/payin-quotes` applies for MXN, COP, and ARS sender amounts |
| `currency_type` | `sender` or `receiver`, same meaning as on `/payin-quotes` |

`/fx` only previews a payin direction: `to` must be a stablecoin and `from` must be a fiat currency. Passing anything else fails with `VALIDATION_INVALID_REQUEST`.

### Response

| Field | Notes |
| --- | --- |
| `commercial_quotation` | Raw market exchange rate |
| `blindpay_quotation` | Exchange rate including BlindPay's fee |
| `result_amount` | The other side of the conversion: the sender amount if `currency_type` is `receiver`, otherwise the receiver amount |
| `instance_flat_fee` | Flat-fee component |
| `instance_percentage_fee` | Percentage fee in basis points (0-10000 = 0-100%) |

There is no `partner_fee_id` parameter on this endpoint, so `instance_percentage_fee` never includes a partner fee markup, even if you'd normally pass one to `/payin-quotes`.

## OTC (Over-the-Counter) payin quotes

Set `is_otc: true` to price the quote through BlindPay's OTC desk instead of the standard rate. OTC is scoped narrowly:

| Constraint | Detail |
| --- | --- |
| Sender currency | BRL only. Any other currency fails with `QUOTES_OTC_NOT_SUPPORTED` |
| Token | USDT only. `token: "USDC"` fails with `QUOTES_OTC_NOT_SUPPORTED` |
| `currency_type` / `cover_fees` | Only `currency_type: "sender"` with `cover_fees: false`, or `currency_type: "receiver"` with `cover_fees: true`. Any other combination fails with `QUOTES_OTC_NOT_SUPPORTED` |
| Minimum amount | $5,000 USD equivalent. Below that, the quote fails with `PAYOUTS_AMOUNT_BELOW_MINIMUM` |
| Quote expiry | **10 seconds**, not 5 minutes. See [cut-off times](../kb/cut-off-times.md) |

```bash [cURL]
curl https://api.blindpay.com/v1/instances/in_000000000000/payin-quotes \
  --request POST \
  --header 'Content-Type: application/json' \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --data '{
  "blockchain_wallet_id": "bw_000000000000",
  "currency_type": "sender",
  "cover_fees": false,
  "request_amount": 3000000,
  "payment_method": "pix",
  "token": "USDT",
  "is_otc": true
}'
```

On an OTC quote, `blindpay_quotation` equals `commercial_quotation`: fees show up in `flat_fee` and `partner_fee_amount` instead of being baked into the exchange rate.

## Testing

On development instances, the amount you request determines the outcome once the resulting payin is created:

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

| Amount | Result |
| --- | --- |
| 666.00 | Failed |
| 777.00 | Refunded |
| Any other amount | Completes automatically, about 30 seconds after initiation |

**Advanced flavor (stablecoin and blockchain mechanics)**

| Amount | Result |
| --- | --- |
| 666.00 | Failed |
| 777.00 | Refunded |
| Any other amount | Completes automatically, about 30 seconds after initiation, and delivers `USDB` to the destination wallet on the matching testnet |

## Related

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

- [Payins](payins.md): create the payin from a quote and track it to completion
- [Virtual accounts](../virtual-accounts/virtual-accounts.md): an alternative destination that skips the memo-code flow
- [Partner fees](../essentials/partner-fees.md): how `partner_fee_id` is calculated and paid out
- [Cut-off times](../kb/cut-off-times.md): settlement windows by payment method

**Advanced flavor (stablecoin and blockchain mechanics)**

- [Payins](payins.md): create the payin from a quote and track the on-chain delivery to completion
- [Blockchain wallets](blockchain-wallets.md): register the external wallet a payin quote can target
- [Wallets](../wallets/wallets.md): the BlindPay-managed wallet alternative to a blockchain wallet
- [Supported chains](../kb/supported-chains.md): full chain and token compatibility matrix
- [Partner fees](../essentials/partner-fees.md): how `partner_fee_id` is calculated and paid out
