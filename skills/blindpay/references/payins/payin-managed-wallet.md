# Payin with managed wallet

Accept a fiat payment and have the equivalent stablecoins delivered into a BlindPay-managed wallet, two REST calls with no signing.

Source: https://blindpay.com/docs/payin-managed-wallet

This is the simplest way to receive a [payin](payins.md): the sender pays fiat, and BlindPay delivers the equivalent stablecoins into a [managed wallet](../wallets/wallets.md). BlindPay generates the wallet address and custodies the balance, so there is no external wallet to connect and nothing to sign. You create a payin quote and create the payin, two REST calls.

If the stablecoins should be delivered to a wallet the customer controls instead, see [Payin with blockchain wallet](payin-blockchain-wallet.md).

## Prerequisites

**Before you start:**

1. Create an account at https://app.blindpay.com/sign-up
2. Create a development instance (see essentials/instances.md)
3. Create your API key (see essentials/api-keys.md)

You also need:

- A [customer](../essentials/customers.md) with `kyc_status: "approved"`
- A [managed wallet](../wallets/wallets.md) (`bl_...`) for that customer as the delivery target

### Create a payin quote

A payin quote locks in how much fiat the sender sends and how much the wallet receives. Pass `wallet_id` to target the managed wallet; BlindPay detects the delivery network from the wallet, so you never pass a network on a payin quote. This example quotes an ACH payin with the sender covering the fee, so `request_amount` is the amount the sender sends.

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID.

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/payin-quotes \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "wallet_id": "bl_000000000000",
    "currency_type": "sender",
    "cover_fees": true,
    "request_amount": 10000,
    "payment_method": "ach",
    "token": "USDB"
  }'
```

`request_amount` is an integer in minor units, so `10000` here means $100.00. On development instances the token is always `USDB`; in production use `USDC` or `USDT`. Save the quote ID (`qu_...`); you have 5 minutes to create the payin before it expires.

### Create the payin

Create the payin from the quote ID. This generates the payment instructions the sender pays into.

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/payins/evm \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "payin_quote_id": "qu_000000000000"
  }'
```

The response carries the instructions matching the quote's `payment_method`: `memo_code` and `blindpay_bank_details` for ACH and wire, `pix_code` for Pix, `clabe` for SPEI. Share them with the sender.

## What happens next

Once the fiat deposit clears, BlindPay converts it and credits the managed wallet. Two webhooks confirm it: `payin.complete` for the payin itself and `wallet.inbound` when the stablecoins land in the wallet. On a development instance the payin settles automatically about 30 seconds after creation.

## Related

- [Payins](payins.md): the payin reference, statuses, testing sentinels, and webhooks
- [Payin quotes](payin-quotes.md): full request and response fields
- [Managed wallets](../wallets/wallets.md): the delivery wallet, including the balance endpoint
- [Payin with blockchain wallet](payin-blockchain-wallet.md): deliver to a customer-controlled wallet instead
