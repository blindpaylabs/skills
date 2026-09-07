# Bank transfer to stablecoins

Accept a bank transfer and have BlindPay deliver the equivalent stablecoins automatically on a development instance, using only the REST API.

Source: https://blindpay.com/docs/quickstart-payin

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

This guide walks through the minimum steps to receive a fiat payment: accept the terms of service, create a customer, give that customer a stablecoin delivery target, quote the payin, and create it. On a development instance the deposit settles automatically about 30 seconds after you create the payin.

**Advanced flavor (stablecoin and blockchain mechanics)**

This quickstart walks through an on-ramp payin: a sender pays fiat over bank rails and BlindPay delivers the equivalent stablecoins into a managed wallet. You will accept the terms of service, create a customer, create a managed wallet, quote the payin, and create it. Every step is a REST call, no on-chain signing involved. On a development instance the deposit settles automatically about 30 seconds after you create the payin.

<CLLMPrompt src="prompts/quickstart-payin.md" displayText="Guided: the agent walks you through each step." />

<CLLMPrompt src="prompts/quickstart-payin-autonomous.md" displayText="Autonomous: builds everything, one handoff at the end." />

## Before you begin

You need a BlindPay account and an API key for a development instance. See [Instances](../essentials/instances.md) and [API keys](../essentials/api-keys.md).

**Before you start:**

1. Create an account at https://app.blindpay.com/sign-up
2. Create a development instance (see essentials/instances.md)
3. Create your API key (see essentials/api-keys.md)

### Accept terms of service

Every instance requires a terms of service acceptance before you can create customers. For testing, you can accept on behalf of the customer; in production, your customer should accept the terms themselves.

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID.

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/e/instances/in_000000000000/tos \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "idempotency_key": "<your_uuid>"
  }'
```

Open the URL from the response in your browser, accept the terms, and copy the `tos_id` shown on the confirmation screen. You will pass this `tos_id` when you create the customer in the next step.

![Copying the tos_id after accepting the BlindPay terms of service](https://pub-4fabf5dd55154f19a0384b16f2b816d9.r2.dev/blindpay_tos_acceptance-min.jpg)

### Create a customer

Every payment flows through a customer that has completed KYC. This example creates an individual customer with standard KYC.

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID.

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "tos_id": "to_000000000000",
    "type": "individual",
    "kyc_type": "standard",
    "email": "email@example.com",
    "tax_id": "12345678",
    "address_line_1": "8 The Green",
    "address_line_2": "#12345",
    "city": "Dover",
    "state_province_region": "DE",
    "country": "US",
    "postal_code": "02050",
    "ip_address": "127.0.0.1",
    "phone_number": "+13022006100",
    "proof_of_address_doc_type": "UTILITY_BILL",
    "proof_of_address_doc_file": "https://example.com/proof-of-address.jpg",
    "first_name": "John",
    "last_name": "Doe",
    "date_of_birth": "1998-01-01T00:00:00Z",
    "id_doc_country": "US",
    "id_doc_type": "PASSPORT",
    "id_doc_front_file": "https://example.com/passport-front.jpg",
    "selfie_file": "https://example.com/selfie.jpg"
  }'
```

Save the `id` from the response: this is your customer ID (`re_...`).

**Note:**

On development instances, KYC is approved automatically. In production, automated review for standard KYC takes about 60 seconds.

### Create a managed wallet

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

A payin needs somewhere to deliver the stablecoins once the fiat arrives, so it asks for a wallet ID rather than a bank detail. The simplest destination is a [managed wallet](../wallets/wallets.md): BlindPay generates the address and custodies the balance, so there is no external wallet to connect and nothing to sign.

**Advanced flavor (stablecoin and blockchain mechanics)**

The payin needs somewhere to deliver the stablecoins once the fiat arrives. A [managed wallet](../wallets/wallets.md) is the simplest destination: BlindPay generates the address and custodies the balance, so there is nothing to sign and no external wallet to connect.

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID, `re_000000000000` with your customer ID.

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/wallets \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "network": "base_sepolia",
    "name": "Quickstart Wallet"
  }'
```

Save the `id` from the response: this is your managed wallet ID (`bl_...`).

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

This wallet is settlement plumbing, not the part you build a UI around. If your customer wants the stablecoins delivered to a wallet they control instead, register a [blockchain wallet](../payins/blockchain-wallets.md) and pass its `blockchain_wallet_id` on the quote. And if you want deposits to arrive without memo codes, create a [virtual account](../virtual-accounts/virtual-accounts.md) for the customer.

**Advanced flavor (stablecoin and blockchain mechanics)**

If your customer wants stablecoins delivered to a wallet they control instead, register a [blockchain wallet](../payins/blockchain-wallets.md) and use its `blockchain_wallet_id` on the quote in the next step.

### Create a payin quote

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

A payin quote locks in how much fiat the sender sends and how much the customer receives before you create the payin. Pass `wallet_id` to target the managed wallet; BlindPay detects the delivery network from the wallet. This example quotes an ACH payin with the sender covering the fee, so `request_amount` is the amount the sender sends.

**Advanced flavor (stablecoin and blockchain mechanics)**

A payin quote locks in how much fiat the sender sends and how much the wallet receives before you create the payin. Pass `wallet_id` to target the managed wallet; BlindPay detects the delivery network from the wallet, so you never pass a network on a payin quote. This example quotes an ACH payin with the sender covering the fee, so `request_amount` is the amount the sender sends.

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

`request_amount` is an integer in minor units, so `10000` here means $100.00. On development instances the token is always `USDB`, BlindPay's test stablecoin.

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

In production use `USDC` or `USDT`.

Save the `id` from the response: this is your payin quote ID (`pq_...`). You have 5 minutes to create the payin before the quote expires.

**Note:**

This example uses the default manual bank transfer. If the customer connected a bank account through [Plaid](../payouts/bank-accounts.md#connect-with-plaid), use `payment_method: "ach_pull"` with the account id in `funding_bank_account_id` instead and BlindPay pulls the funds automatically; see [Payins](../payins/payins.md#pull-funding-from-a-plaid-connected-account).

### Create the payin

Create the payin from the quote ID. This is what generates the bank details your end user pays into.

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/payins/evm \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "payin_quote_id": "pq_000000000000"
  }'
```

The response includes `memo_code` and `blindpay_bank_details`. Share these with the end user so they know where to send the ACH transfer and how to reference it.

```json [Response]
{
  // ...
  "memo_code": "12345678",
  "blindpay_bank_details": {
    "routing_number": "121145349",
    "account_number": "621327727210181",
    "account_type": "Business checking",
    "beneficiary": {
      "name": "BlindPay, Inc.",
      "address_line_1": "8 The Green, #19364",
      "address_line_2": "Dover, DE 19901"
    },
    "receiving_bank": {
      "name": "Example Bank, N.A.",
      "address_line_1": "1 Example Plaza",
      "address_line_2": "San Francisco, CA 94129"
    }
  }
  // ...
}
```

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

**Note:**

If the customer has an enabled virtual account, the `memo_code` is ignored and the deposit goes straight to their dedicated account details instead. See [Virtual accounts](../virtual-accounts/virtual-accounts.md).

## What happens next

On a development instance, the payin settles automatically about 30 seconds after you create it. Two webhooks confirm it: `payin.complete` for the payin itself and `wallet.inbound` when the stablecoins land in the managed wallet.

**Advanced flavor (stablecoin and blockchain mechanics)**

You can also check the wallet balance directly:

```bash [cURL]
curl --request GET \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/wallets/bl_000000000000/balance \
  --header 'Authorization: Bearer YOUR_API_KEY'
```

In production, BlindPay waits up to 5 business days for an ACH or wire deposit to arrive before cancelling the payin if nothing shows up.

**Success:**

Done. To receive another payment, create a new payin quote and repeat the last step.

## Related

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

- [Bank transfer in](../payins/payins.md): full reference for every payment method, including Pix, SPEI, and PSE
- [Virtual accounts](../virtual-accounts/virtual-accounts.md): a dedicated deposit account whose deposits settle to the linked wallet
- [Pay out to bank](../payouts/payouts.md): send funds back out to a bank account
- [Webhooks](../essentials/webhooks.md): handle payin and customer events in real time

**Advanced flavor (stablecoin and blockchain mechanics)**

- [Payins](../payins/payins.md): full reference for every payment method, including Pix, SPEI, and PSE
- [Managed wallet](../wallets/wallets.md): the wallet entity this quickstart delivers to
- [Stablecoins to bank transfer](quickstart-payout.md): the payout counterpart of this guide
- [Webhooks](../essentials/webhooks.md): handle `payin.complete` and `wallet.inbound` in real time
