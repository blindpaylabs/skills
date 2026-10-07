# Partner fees

Add percentage or flat fees to your customers' transactions and withdraw the accumulated revenue monthly.

Source: https://blindpay.com/docs/learn/partner-fees

## What it is

A partner fee is a markup you add on top of transactions processed through BlindPay. BlindPay collects the fee from your customer during each transaction, accumulates it over the calendar month, and releases the balance to you for withdrawal on the first day of the following month.

## Fee types

You can configure two types of fees, set independently for payins and payouts:

| Type | Description |
| --- | --- |
| Percentage | A percentage of the transaction amount |
| Flat | A fixed amount per transaction |

## Monthly collection cycle

Partner fees follow a monthly collection and withdrawal cycle:

- Fees are collected automatically from your customer during each transaction, throughout the month.
- All collected fees accumulate over the calendar month.
- On the first day of the following month, the total balance is released and becomes available for withdrawal.
- BlindPay nets your outstanding invoice out of the accumulated fees first. You never pay your BlindPay invoice separately; you receive the net amount.

**Note:**

`payin.partnerFee` and `payout.partnerFee` are declared in the webhook catalog but do not currently fire, and there is no `tracking_partner_fee` object on quote or transaction responses. The amount collected on a given transaction is `partner_fee_amount` on that quote (see [Quote response fields](#quote-response-fields) below); there is no separate real-time event or status object for partner fee collection today.

## Configure a partner fee

### Create a fee configuration

Create a fee configuration for payins, payouts, or both:

| Field | Range | Meaning |
| --- | --- | --- |
| `payin_percentage_fee`, `payout_percentage_fee` | 0-9900 | Basis points; `100` means 1%. Capped at 99% of the transaction amount. |
| `payin_flat_fee`, `payout_flat_fee` | 0-100000 | Minor units (cents); `200` means $2.00. Capped at $1,000.00. |

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/partner-fees \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "name": "Display Name",
    "payin_percentage_fee": 100,
    "payin_flat_fee": 0,
    "payout_percentage_fee": 0,
    "payout_flat_fee": 200
  }'
```

Save the `id` from the response: this is your partner fee ID (`pf_...`). You can also manage fee configurations from the [BlindPay dashboard](https://app.blindpay.com), under the instance's Partner Fees tab. `POST /partner-fees` and `DELETE /partner-fees/{id}` accept an `Idempotency-Key` header to safely retry the request.

To make a fee the default for all [virtual account](../virtual-accounts/virtual-accounts.md) deposits, set `virtual_account_set: true` on creation (only one active fee per instance can hold the flag; defaults to `false` when omitted). Send it as an actual boolean, not a string: any non-empty string value, including `"false"`, is coerced to `true`. See [Virtual accounts](#virtual-accounts) below for how defaults and per-account fees interact.

**Warning:**

There is no update endpoint. To change a fee's amounts or its `virtual_account_set` flag, delete the existing configuration and create a new one.

### Pass it in your quote requests

Reference the `partner_fee_id` in a payin quote, payout quote, or transfer quote to apply that fee to the transaction.

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/payin-quotes \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "blockchain_wallet_id": "bw_000000000000",
    "currency_type": "sender",
    "cover_fees": false,
    "request_amount": 10000,
    "payment_method": "ach",
    "token": "USDC",
    "partner_fee_id": "pf_000000000000"
  }'
```

```js [index.js]
const response = await fetch(
  'https://api.blindpay.com/v1/instances/in_000000000000/payin-quotes',
  {
    method: 'POST',
    headers: {
      Authorization: 'Bearer YOUR_API_KEY',
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      blockchain_wallet_id: 'bw_000000000000',
      currency_type: 'sender',
      cover_fees: false,
      request_amount: 10000,
      payment_method: 'ach',
      token: 'USDC',
      partner_fee_id: 'pf_000000000000',
    }),
  }
)

const data = await response.json()
```

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID.

## Virtual accounts

Deposits into a [virtual account](../virtual-accounts/virtual-accounts.md) create payins automatically, with no quote request where you could pass a `partner_fee_id`. Instead, the fee is configured ahead of time, at two levels:

1. **Instance-wide default.** Create a fee configuration with `virtual_account_set: true` (or toggle it in the dashboard's Partner Fees tab). It applies to every virtual account deposit on the instance. Only one active configuration can be the default.
2. **Per virtual account.** Pass a `partner_fee_id` when [creating a virtual account](../virtual-accounts/virtual-accounts-create.md), or set it later with the update endpoint. A fee pinned to an account **overrides the instance-wide default** for that account's deposits. Update with `partner_fee_id: null` to clear the pin and fall back to the default.

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/virtual-accounts \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "banking_partner": "cfsb",
    "token": "USDC",
    "blockchain_wallet_id": "bw_000000000000",
    "partner_fee_id": "pf_000000000000"
  }'
```

Deleting a fee configuration automatically unpins it from any virtual accounts referencing it; those accounts fall back to the instance-wide default.

**Note:**

Every virtual account deposit always delivers at least $0.01 on-chain. The partner fee is collected from what remains after any transaction-time BlindPay fee, so on small deposits collection can be partial (a $5.00 fee on a $5.00 deposit collects $4.99) or zero on micro-deposits. See [fees on deposits](../virtual-accounts/virtual-accounts.md#fees-on-deposits).

**Warning:**

A partner fee (pinned or the instance-wide `virtual_account_set` default) is never applied to a virtual account deposit that settles on Solana or Tron. Deposits on those two networks skip partner fee collection entirely, whatever fee is configured.

## Manage fee configurations

```bash [cURL]
curl --url https://api.blindpay.com/v1/instances/in_000000000000/partner-fees \
  --header 'Authorization: Bearer YOUR_API_KEY'
```

```bash [cURL]
curl --url https://api.blindpay.com/v1/instances/in_000000000000/partner-fees/pf_000000000000 \
  --header 'Authorization: Bearer YOUR_API_KEY'
```

```bash [cURL]
curl --request DELETE \
  --url https://api.blindpay.com/v1/instances/in_000000000000/partner-fees/pf_000000000000 \
  --header 'Authorization: Bearer YOUR_API_KEY'
```

- `GET /partner-fees` returns every configuration for the instance in a single array. There's no pagination.
- `GET /partner-fees/{id}` and `POST /partner-fees` return only `id`, `instance_id`, `name`, and the four fee fields. Only the list endpoint also returns `virtual_account_set`, `created_at`, and `updated_at` for each item. List the fees rather than reading the create or single-get response if you need those.
- `DELETE /partner-fees/{id}` always returns `{"success": true}`, even when `id` doesn't exist, belongs to another instance, or was already deleted. The response doesn't tell you whether a row was actually removed.

**Warning:**

`GET /partner-fees/{id}` on an `id` that doesn't exist, belongs to another instance, or was already deleted returns `500 INTERNAL_ERROR`, not `404`. Don't use the status code to detect a missing fee; only look up an `id` you got back from a prior create or list call.

## Quote response fields

Each payin quote, payout quote, and transfer quote response includes:

| Field | Description |
| --- | --- |
| `partner_fee_amount` | Exact amount collected as a partner fee for this transaction |

## Example

A 1% payin fee and a $2.00 flat payout fee configured on the same instance:

| Transaction | Customer pays | Partner fee collected |
| --- | --- | --- |
| $100 payin | $101.00 | $1.00 |
| $100 payout | $102.00 | $2.00 |

Monthly settlement example:

| | Amount |
| --- | --- |
| Total partner fees collected in January | $500.00 |
| BlindPay invoice for January | $250.00 |
| Available for withdrawal on February 1 | $250.00 |

## Related

- [Webhooks](webhooks.md): full event reference and signature verification
- [Billing](billing.md): how BlindPay invoices your account
- [Fiat receive](../payins/payins.md): payin quotes that accept `partner_fee_id`
- [Fiat send](../payouts/payouts.md): payout quotes that accept `partner_fee_id`
- [Stablecoin send](../wallets/send.md): transfer quotes that accept `partner_fee_id`
- [Virtual accounts](../virtual-accounts/virtual-accounts.md): automated deposits with default or per-account partner fees
