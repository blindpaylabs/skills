# Stablecoins to bank transfer

Send your first off-ramp payout from a BlindPay-managed wallet to a bank account on a development instance, using only the REST API.

Source: https://blindpay.com/docs/quickstart-payout

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

This quickstart walks through a payout: funds leave a BlindPay-managed wallet and USD lands in a recipient's bank account. You will accept the terms of service, create a customer, create and fund a managed wallet, add a bank account, quote the payout, and execute it. Because BlindPay custodies the funding wallet, there is no on-chain signing or token approval; every step is a REST call. On a development instance the payout completes automatically a few seconds after you execute it.

<CLLMPrompt src="prompts/quickstart-payout.md" displayText="Guided: the agent walks you through each step." />

<CLLMPrompt src="prompts/quickstart-payout-autonomous.md" displayText="Autonomous: builds everything, one handoff at the end." />

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
    "idempotency_key": "<your_uuid>",
    "redirect_url": "https://example.com/tos-accepted"
  }'
```

Open the URL from the response in your browser and accept the terms. The page itself does not display the `tos_id` (`to_...`): after you accept, the browser lands on your `redirect_url` with `?tos_id=to_...` in the address bar. It is also in the response of the `tos` request in your browser's DevTools Network tab, as shown below, and in the `tos.accept` webhook. You will pass this `tos_id` when you create the customer in the next step.

![Copying the tos_id from the tos request in DevTools after accepting the BlindPay terms of service](https://pub-4fabf5dd55154f19a0384b16f2b816d9.r2.dev/blindpay_tos_acceptance-min.jpg)

### Create a customer

Every payout requires a customer that has completed KYC. This example creates an individual customer with standard KYC.

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

Every payout needs a funding source, the wallet the settlement stablecoins are pulled from. A [managed wallet](../wallets/wallets.md) is the simplest one: BlindPay generates the address and holds the keys, so executing the payout later needs no approval or signature. This example creates it on Solana Devnet, one of the development networks.

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID, `re_000000000000` with your customer ID.

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/wallets \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "network": "solana_devnet",
    "name": "Quickstart Wallet"
  }'
```

Save the `id` (`bl_...`) and the `address` from the response. The `id` targets the payin in the next step; the `address` executes the payout.

### Fund the wallet with a payin

A [payin](../payins/payins.md) funds the wallet: fiat comes in and BlindPay delivers USDB, the development-only test stablecoin, into it. Create a payin quote first, passing the wallet's `id`:

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

`request_amount` is an integer in minor units, so `10000` is $100.00. Save the quote ID (`qu_...`) and create the payin from it within 5 minutes:

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/payins/evm \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "payin_quote_id": "qu_000000000000"
  }'
```

On a development instance the payin completes automatically about 30 seconds after creation; no real fiat moves. The wallet then holds about 100 USDB to pay out from.

### Add a bank account

This is the payout destination, the bank account that receives the USD. This example adds a US ACH account. Use real, valid bank details, even on development.

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID, `re_000000000000` with your customer ID.

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/bank-accounts \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "type": "ach",
    "name": "Display Name",
    "beneficiary_name": "<Replace this>",
    "routing_number": "<Replace this>",
    "account_number": "<Replace this>",
    "account_type": "checking",
    "account_class": "individual",
    "recipient_relationship": "other",
    "address_line_1": "123 Main St",
    "city": "New York",
    "state_province_region": "NY",
    "country": "US",
    "postal_code": "10001"
  }'
```

Save the bank account ID (`ba_...`).

### Create a payout quote

A quote locks the conversion rate and fees for 5 minutes. The `network` and `token` describe the funding wallet: `solana_devnet` and `USDB` for the wallet you just funded.

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/quotes \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "bank_account_id": "ba_000000000000",
    "currency_type": "sender",
    "cover_fees": false,
    "request_amount": 5000,
    "network": "solana_devnet",
    "token": "USDB"
  }'
```

`request_amount` is an integer in minor units, so `5000` here means $50.00. `cover_fees: false` means the fee is deducted from what the bank account receives, the common case. Save the quote ID (`qu_...`).

### Execute the payout

Execute the payout by passing the quote ID and the managed wallet's address as the funding source. Because BlindPay custodies the wallet, there is no approve or delegation step; this single call moves the funds.

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

The response returns the payout with `status: "processing"`.

**Note:**

The endpoint is `/payouts/evm` regardless of the funding network; it handles managed wallets on every supported chain.

## What happens next

On a development instance, the payout completes automatically a few seconds after you execute it. Check for the `payout.complete` webhook to confirm. In production, settlement timing depends on the payout rail; see [cut-off times](../kb/cut-off-times.md).

**Success:**

Done. To send another payment, create a new payout quote and execute it again; each quote is single-use.

## Related

- [Pay out to bank](../payouts/payouts.md): full reference for every payout rail, including Pix, SPEI, and SWIFT
- [Bank transfer to stablecoins](quickstart-payin.md): the payin counterpart of this guide
- [Managed wallet](../wallets/wallets.md): the funding wallet entity used in this quickstart
- [Webhooks](../essentials/webhooks.md): handle `payout.complete` and other events in real time

**Advanced flavor (stablecoin and blockchain mechanics)**

This quickstart walks through an off-ramp payout: pulling USDB from a managed wallet and delivering USD to a bank account. You will accept the terms of service, create a customer, create and fund a managed wallet, add a bank account, create a payout quote, and execute the payout. Every step is a REST call, no on-chain signing involved.

If you want to fund the payout from a self-custodied wallet instead (with the on-chain authorization that entails), follow the [Payout with EVM](../payouts/payout-evm.md), [Stellar](../payouts/payout-stellar.md), or [Solana](../payouts/payout-solana.md) tutorials after this one.

<CLLMPrompt src="prompts/quickstart-payout.md" displayText="Guided: the agent walks you through each step." />

<CLLMPrompt src="prompts/quickstart-payout-autonomous.md" displayText="Autonomous: builds everything, one handoff at the end." />

## Before you begin

You need a BlindPay account and an API key for a development instance. See [Instances](../essentials/instances.md) and [API keys](../essentials/api-keys.md).

**Before you start:**

1. Create an account at https://app.blindpay.com/sign-up
2. Create a development instance (see essentials/instances.md)
3. Create your API key (see essentials/api-keys.md)

## Steps

### Accept terms of service

For testing you can accept the terms of service yourself. In production, your customer should be the one accepting them.

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID.

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/e/instances/in_000000000000/tos \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "idempotency_key": "<your_uuid>",
    "redirect_url": "https://example.com/tos-accepted"
  }'
```

Open the URL from the response in your browser and accept the terms. The page itself does not display the `tos_id` (`to_...`): after you accept, the browser lands on your `redirect_url` with `?tos_id=to_...` in the address bar. It is also in the response of the `tos` request in your browser's DevTools Network tab, as shown below, and in the `tos.accept` webhook. You will need it to create the customer.

![Copying the tos_id from the tos request in DevTools after accepting the BlindPay terms of service](https://pub-4fabf5dd55154f19a0384b16f2b816d9.r2.dev/blindpay_tos_acceptance-min.jpg)

### Create a customer

Every payout requires a customer that has completed KYC. On development instances, customers are auto-approved.

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
    "phone_number": "+1234567890",
    "proof_of_address_doc_type": "UTILITY_BILL",
    "proof_of_address_doc_file": "https://pub-4fabf5dd55154f19a0384b16f2b816d9.r2.dev/v4-460px-Get-Proof-of-Address-Step-3-Version-2.jpg.jpeg",
    "first_name": "John",
    "last_name": "Doe",
    "date_of_birth": "1998-01-01T00:00:00Z",
    "id_doc_country": "US",
    "id_doc_type": "PASSPORT",
    "id_doc_front_file": "https://pub-4fabf5dd55154f19a0384b16f2b816d9.r2.dev/1000_F_365165797_VwQbNaD4yjWwQ6y1ENKh1xS0TXauOQvj.jpg",
    "selfie_file": "https://pub-4fabf5dd55154f19a0384b16f2b816d9.r2.dev/selfie.png"
  }'
```

This is the full standard KYC payload (`kyc_type: "standard"`): identity document, selfie, and proof of address. Save the customer ID (`re_...`) from the response.

### Create a managed wallet

The payout needs a funding source, the wallet the stablecoins are pulled from. A [managed wallet](../wallets/wallets.md) is the simplest one: BlindPay generates the address and holds the keys, so executing the payout later needs no approval or signature. This example creates it on Solana Devnet, one of the development networks.

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID, `re_000000000000` with your customer ID.

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/wallets \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "network": "solana_devnet",
    "name": "Quickstart Wallet"
  }'
```

Save the `id` (`bl_...`) and the `address` from the response. The `id` targets the payin in the next step; the `address` executes the payout.

### Fund the wallet with a payin

A [payin](../payins/payins.md) funds the wallet: fiat comes in and BlindPay delivers USDB, the development-only test stablecoin, into it. Create a payin quote first, passing the wallet's `id`:

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

`request_amount` is an integer in minor units, so `10000` is $100.00. Save the quote ID (`qu_...`) and create the payin from it within 5 minutes:

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/payins/evm \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "payin_quote_id": "qu_000000000000"
  }'
```

On a development instance the payin completes automatically about 30 seconds after creation; no real fiat moves. The wallet then holds about 100 USDB to pay out from.

### Add a bank account

This is the fiat destination for the payout, the bank account that receives the USD once the stablecoin is converted. This example adds a US ACH account. Use real, valid bank details, even on development.

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID, `re_000000000000` with your customer ID.

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/bank-accounts \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "type": "ach",
    "name": "Display Name",
    "beneficiary_name": "<Replace this>",
    "routing_number": "<Replace this>",
    "account_number": "<Replace this>",
    "account_type": "checking",
    "account_class": "individual",
    "recipient_relationship": "other",
    "address_line_1": "123 Main St",
    "city": "New York",
    "state_province_region": "NY",
    "country": "US",
    "postal_code": "10001"
  }'
```

Save the bank account ID (`ba_...`).

### Create a payout quote

A quote locks the conversion rate and fees for 5 minutes. The `network` and `token` describe the funding wallet: `solana_devnet` and `USDB` for the wallet you just funded.

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/quotes \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "bank_account_id": "ba_000000000000",
    "currency_type": "sender",
    "cover_fees": false,
    "request_amount": 5000,
    "network": "solana_devnet",
    "token": "USDB"
  }'
```

`request_amount` is an integer in minor units, `5000` is $50.00. `cover_fees: false` means the fee is deducted from what the bank account receives, the common case. Save the quote ID (`qu_...`).

### Execute the payout

Execute the payout by passing the quote ID and the managed wallet's address as the funding source. Because BlindPay custodies the wallet, there is no `approve` call, signature, or delegation; this single call moves the funds.

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

The response returns the payout with `status: "processing"`. On a development instance, the payout completes automatically a few seconds after creation, no manual settlement step needed. You will receive a `payout.complete` webhook once it does.

**Success:**

Congratulations! You've completed your first off-ramp payout with BlindPay. To send another, create a new quote and execute again, each quote is single-use.

## Next steps

This quickstart used a managed wallet, so no on-chain authorization was needed. To pull funds from a self-custodied wallet instead, follow the per-network tutorials: [EVM](../payouts/payout-evm.md) (ERC-20 approve), [Stellar](../payouts/payout-stellar.md) (authorize + signed XDR), or [Solana](../payouts/payout-solana.md) (token delegation).

## Related

- [Payouts](../payouts/payouts.md): the full payout section, with a tutorial per funding path
- [Payins](../payins/payins.md): accept fiat and deliver stablecoins to a wallet
- [Managed wallet](../wallets/wallets.md): the funding wallet entity used in this quickstart
- [Webhooks](../essentials/webhooks.md): handle `payout.complete` and other real-time events
- [KYC](../kb/kyc.md): customer verification levels and required fields
