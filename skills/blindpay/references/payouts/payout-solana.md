# Payout with Solana

Fund a payout from an external Solana wallet, delegate the tokens to BlindPay, then create the payout.

Source: https://blindpay.com/docs/payout-solana

This tutorial funds a [payout](payouts.md) from an external [blockchain wallet](../payins/blockchain-wallets.md) on Solana. Solana payouts need a token delegation before the payout executes: the sender delegates the quoted amount to BlindPay, then BlindPay pulls it during payout processing.

If the funds live in a BlindPay-custodied wallet instead, skip the delegation entirely: see [Payout with managed wallet](payout-managed-wallet.md).

**Note:**

The examples below use `solana_devnet` and `USDB` since they're the development network and test token. Swap in the production network and token (`USDC` or `USDT`) when you go live.

## Prerequisites

**Before you start:**

1. Create an account at https://app.blindpay.com/sign-up
2. Create a development instance (see essentials/instances.md)
3. Create your API key (see essentials/api-keys.md)

You also need a [customer](../essentials/customers.md) with `kyc_status: "approved"`, a [bank account](bank-accounts.md) (`ba_...`), and an unexpired [payout quote](payout-quotes.md) (`qu_...`) created with a Solana `network`.

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID, `re_000000000000` with your customer ID, `ba_000000000000` with your bank account ID.

### Prepare the delegation transaction

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/prepare-delegate-solana \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "owner_address": "YOUR_SOLANA_WALLET_ADDRESS",
    "quote_id": "qu_000000000000"
  }'
```

With `quote_id`, BlindPay takes the token mint and the quoted `sender_amount` from the quote. Without it, send `token_address` (the SPL mint) and `amount` as a decimal token amount (`"10.5"` for 10.5 USDC), not base units.

This returns a serialized transaction to sign:

```json
{
  "success": true,
  "transaction": "AQAAA..."
}
```

### Sign and submit the delegation

Install the Solana dependencies:

```bash
npm install @solana/web3.js bs58
```

```js [index.js]
import { Connection, VersionedTransaction } from '@solana/web3.js'
import { Buffer } from 'node:buffer'

const RPC_URL = 'https://api.devnet.solana.com' // Use a mainnet RPC in production
const connection = new Connection(RPC_URL, 'confirmed')

// Serialized transaction from the previous step
const serializedTransaction = 'AQAAA...'

async function signAndSubmitDelegation() {
  const transactionBuffer = Buffer.from(serializedTransaction, 'base64')
  const transaction = VersionedTransaction.deserialize(transactionBuffer)

  // Sign with the sender's wallet (a browser wallet like Phantom, or a local Keypair server-side)
  const signedTransaction = await window.solana.signTransaction(transaction)

  const signature = await connection.sendTransaction(signedTransaction)
  await connection.confirmTransaction(signature, 'confirmed')

  console.log('Delegation signature:', signature)
  return signature
}

signAndSubmitDelegation()
```

Run it:

```bash
node solana-delegate.js
```

### Create the payout

Once the delegation transaction confirms, create the payout at `/payouts/solana`. Pass the delegation signature from the previous step as `signed_transaction`: it is required on production instances, where BlindPay screens it before pulling the funds.

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/payouts/solana \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
  "quote_id": "qu_000000000000",
  "sender_wallet_address": "YOUR_SOLANA_WALLET_ADDRESS",
  "signed_transaction": "DELEGATION_SIGNATURE"
}'
```

**Note:**

Repeat the delegation (prepare, sign, submit) for every Solana payout. Each delegation only authorizes the amount tied to that specific quote.

## Related

- [Payouts](payouts.md): status lifecycle, cover_fees, testing sentinels, and webhooks
- [Payout quotes](payout-quotes.md): full request and response fields
- [Payin with blockchain wallet](../payins/payin-blockchain-wallet.md): fund your test wallet on development instances
- [Payout with managed wallet](payout-managed-wallet.md): the no-signing alternative
