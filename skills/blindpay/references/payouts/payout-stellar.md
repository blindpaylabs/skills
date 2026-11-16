# Payout with Stellar

Fund a payout from an external Stellar wallet, authorize the payout, sign the XDR transaction, then create the payout.

Source: https://blindpay.com/docs/payout-stellar

This tutorial funds a [payout](payouts.md) from an external [blockchain wallet](../payins/blockchain-wallets.md) on Stellar. Stellar has no allowance mechanism, so the client signs a real payment transaction to BlindPay's treasury address and BlindPay submits it: authorize, sign, then create the payout.

**Note:**

BlindPay's Stellar mainnet treasury address: `GCOSSQDM2SWMHRP7CDBOLL2V45NHCRLUWUCEHPPBA2ABCOOLPOLZKIHE`. This is the address that receives payout crypto and sends payin crypto on Stellar mainnet.

**Note:**

The examples below use `stellar_testnet` and `USDB` since they're the development network and test token. Swap in the production network and token (`USDC`) when you go live.

## Prerequisites

**Before you start:**

1. Create an account at https://app.blindpay.com/sign-up
2. Create a development instance (see essentials/instances.md)
3. Create your API key (see essentials/api-keys.md)

You also need a [customer](../essentials/customers.md) with `kyc_status: "approved"`, a [bank account](bank-accounts.md) (`ba_...`), and an unexpired [payout quote](payout-quotes.md) (`qu_...`) created with a Stellar `network`.

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID, `re_000000000000` with your customer ID, `ba_000000000000` with your bank account ID.

### Authorize the payout

Call the authorize endpoint with the quote and the sender's wallet address. This does not consume the quote or create any record. It returns the unsigned transaction envelope (XDR) for the client to sign. The field is named `transaction_hash`, but it holds the envelope, not a hash.

The envelope pays the quote's exact amount to the treasury address above, uses the base fee and is valid for 30 seconds. If it expires before you create the payout, call authorize again and sign the fresh envelope while the quote is still valid.

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/payouts/stellar/authorize \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
  "quote_id": "qu_000000000000",
  "sender_wallet_address": "YOUR_WALLET_ADDRESS"
}'
```

The response is the unsigned envelope, ready to sign:

```json
{
  "transaction_hash": "AAA...AAA"
}
```

### Sign the transaction

Install the Stellar SDK:

```bash
npm install @stellar/stellar-sdk
```

Sign the returned envelope with the sender's Stellar key. Do not submit it to the network yourself: BlindPay submits it when you create the payout, after validating it against the quote. If the create is refused, the funds never leave the wallet.

```js [index.js]
import {
  Keypair,
  Networks,
  TransactionBuilder,
} from '@stellar/stellar-sdk'

const sourceKeypair = Keypair.fromSecret(process.env.STELLAR_SECRET_KEY)

// Rebuild the transaction returned by the authorize endpoint
const transaction = TransactionBuilder.fromXDR(
  transactionHash,
  Networks.TESTNET
)

// Sign it with the sender's key
transaction.sign(sourceKeypair)

// The signed envelope goes in signed_transaction when you create the payout
const signedTransaction = transaction.toXDR()
```

### Create the payout

Create the payout with the quote, the sender's wallet address and the signed envelope from the previous step as `signed_transaction`. BlindPay validates the envelope against the quote (destination, asset and amount), submits it to Stellar and then dispatches the payout. Fee-bumped envelopes are accepted.

Send an `Idempotency-Key` header so a retry after a timeout returns the original payout instead of creating a second one.

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/payouts/stellar \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --header 'Idempotency-Key: YOUR_UNIQUE_KEY' \
  --data '{
  "quote_id": "qu_000000000000",
  "signed_transaction": "YOUR_SIGNED_XDR_ENVELOPE",
  "sender_wallet_address": "YOUR_WALLET_ADDRESS"
}'
```

**Success:**

That's a completed payout on Stellar. To send another, create a new quote and repeat the authorize, sign, and create steps, the quote and signed transaction are single-use.

## Related

- [Payouts](payouts.md): status lifecycle, cover_fees, testing sentinels, and webhooks
- [Payout quotes](payout-quotes.md): full request and response fields
- [Payin with blockchain wallet](../payins/payin-blockchain-wallet.md): fund your test wallet on development instances
- [Payout with managed wallet](payout-managed-wallet.md): the no-signing alternative (Stellar not yet supported for managed wallets)
