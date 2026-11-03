# Payin with blockchain wallet

Accept a fiat payment and have the equivalent stablecoins delivered to a wallet your customer controls.

Source: https://blindpay.com/docs/payin-blockchain-wallet

This tutorial receives a [payin](payins.md) into a [blockchain wallet](blockchain-wallets.md): a self-custodied wallet your customer controls. The flow is the same two REST calls as the managed-wallet path; the difference is that you register the customer's wallet address first, and the stablecoins are delivered on-chain to an address BlindPay does not custody.

If you'd rather hold the balance inside BlindPay, see [Payin with managed wallet](payin-managed-wallet.md).

## Prerequisites

**Before you start:**

1. Create an account at https://app.blindpay.com/sign-up
2. Create a development instance (see essentials/instances.md)
3. Create your API key (see essentials/api-keys.md)

You also need:

- A [customer](../essentials/customers.md) with `kyc_status: "approved"`
- A registered [blockchain wallet](blockchain-wallets.md) (`bw_...`) for that customer, added either by signed message or by direct address

**Warning:**

BlindPay cannot recover stablecoins delivered to a wrong address. Double-check the wallet's `address` and `network` when registering it.

### Create a payin quote

Pass `blockchain_wallet_id` to target the customer's wallet; BlindPay detects the delivery network from the wallet record, so you never pass a network on a payin quote. This example quotes an ACH payin with the sender covering the fee, so `request_amount` is the amount the sender sends.

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID.

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/payin-quotes \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "blockchain_wallet_id": "bw_000000000000",
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

Once the fiat deposit clears, BlindPay converts it and sends the stablecoins on-chain to the registered wallet address, then fires `payin.complete`. Because the wallet is customer-controlled, there is no `wallet.inbound` event; track delivery via the payin's `tracking_complete.transaction_hash` or on a block explorer. On a development instance the payin settles automatically about 30 seconds after creation.

## Related

- [Payins](payins.md): the payin reference, statuses, testing sentinels, and webhooks
- [Payin quotes](payin-quotes.md): full request and response fields
- [Blockchain wallets](blockchain-wallets.md): the signed-message and direct-address registration flows
- [Payin with managed wallet](payin-managed-wallet.md): hold the delivered balance inside BlindPay instead
