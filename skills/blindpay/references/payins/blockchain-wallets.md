# Blockchain wallets

Register an external, customer-controlled wallet address to receive stablecoin payins and send stablecoin payouts.

Source: https://blindpay.com/docs/blockchain-wallets

A blockchain wallet (`bw_...`) is an externally owned wallet that your customer controls, not BlindPay. You register its address so BlindPay can deliver stablecoins to it or read a balance from it, but BlindPay never holds the private keys and cannot move funds out of it without the customer's signature or on-chain authorization.

For a BlindPay-custodied alternative where you do not need the customer to sign anything, see [Managed wallet](../wallets/wallets.md).

**Note:**

Non-custodial by design: BlindPay cannot access, freeze, or recover funds in a blockchain wallet. If a payin or payout is misdirected because of a wrong address, BlindPay cannot reverse the on-chain transfer.

## What it's used for

A blockchain wallet is the endpoint on both sides of a stablecoin movement:

- **Payin delivery target.** On a [payin quote](payin-quotes.md), set `blockchain_wallet_id` to a `bw_...` ID and the stablecoin lands in that wallet once the payin completes.
- **Payout source.** On a [payout](../payouts/payouts.md), the customer authorizes the transfer out of their own blockchain wallet: an on-chain [`approve` on EVM](../payouts/payout-evm.md), a [signed Stellar transaction](../payouts/payout-stellar.md), or a [Solana delegation](../payouts/payout-solana.md), depending on the network.

## Supported networks

| Network | Type | Notes |
| --- | --- | --- |
| `ethereum` | EVM, production | |
| `polygon` | EVM, production | |
| `base` | EVM, production | |
| `arbitrum` | EVM, production | |
| `stellar` | Non-EVM, production | |
| `solana` | Non-EVM, production | |
| `tron` | Non-EVM, production | Beta, requires the `otc` subscription feature on the instance |
| `tempo` | EVM, production | |
| `arc` | EVM, production | |
| `sepolia` | EVM, development | Ethereum testnet |
| `polygon_amoy` | EVM, development | Polygon testnet |
| `base_sepolia` | EVM, development | Base testnet |
| `arbitrum_sepolia` | EVM, development | Arbitrum testnet |
| `stellar_testnet` | Non-EVM, development | |
| `solana_devnet` | Non-EVM, development | |
| `tempo_testnet` | EVM, development | Tempo testnet |
| `arc_testnet` | EVM, development | Arc testnet |

Development instances only accept the testnet networks; production instances only accept the mainnet networks. See [Supported chains](../kb/supported-chains.md) for the full chain and token matrix.

## Prerequisites

**Before you start:**

1. Create an account at https://app.blindpay.com/sign-up
2. Create a development instance (see essentials/instances.md)
3. Create your API key (see essentials/api-keys.md)

A customer must exist before you add a blockchain wallet for them.

**Note:**

Every POST and DELETE call on this page accepts an optional `Idempotency-Key` header. Retrying with the same key and an identical body replays the original response instead of registering or removing the wallet twice; reusing a key with a different body returns an error. Keys are kept for 24 hours.

## Add a blockchain wallet

There are two ways to register a wallet address, controlled by the `is_account_abstraction` field:

- **Signed message (`is_account_abstraction: false`)**, EVM networks only. The customer signs a message with their wallet, and BlindPay recovers the address from the signature server-side, so you never send an address BlindPay has to trust blindly.
- **Direct address (`is_account_abstraction: true`)**, any supported network. You submit the address directly. Despite the field name, this is also how you register Stellar, Solana, and Tron addresses, and how you register EVM smart-contract wallets that cannot produce the standard signature flow.

**Warning:**

Double-check the address before submitting it, especially with the direct-address method. BlindPay cannot verify that an address you paste in actually belongs to your customer, and a stablecoin delivered to the wrong address cannot be recovered.

### Signed message flow

The steps are:

### Get the message to sign

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID, `re_000000000000` with your customer ID.

```bash [cURL]
curl https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/blockchain-wallets/sign-message \
  --header 'Authorization: Bearer YOUR_API_KEY'
```

This returns a fixed message string for the customer to sign. It does not change between requests.

### Sign the message

Use a library like wagmi or ethers.js to sign the message with the customer's wallet and get the signature transaction hash.

```js [ethers.js]
import { signMessage } from '@wagmi/core'

// setup your wagmiConfig
const message = '<returned_from_sign-message_endpoint>'

const signature_tx_hash = await signMessage(wagmiConfig, {
  message,
})
```

### Add the blockchain wallet

Submit the signature. BlindPay recovers the address and stores it, so `address` is omitted from the request body.

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/blockchain-wallets \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "name": "John personal wallet",
    "network": "polygon",
    "is_account_abstraction": false,
    "signature_tx_hash": "0x..."
  }'
```

### Direct address flow

Set `is_account_abstraction: true` and pass the `address` field directly. This is the only option for `stellar`, `solana`, and `tron`, and it also covers EVM smart-contract wallets.

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/blockchain-wallets \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "name": "John personal wallet",
    "network": "polygon",
    "is_account_abstraction": true,
    "address": "0x..."
  }'
```

Each network validates the address format server-side (for example Stellar addresses must start with `G`, Solana addresses are base58, Tron addresses start with `T`), but this is a format check, not proof of ownership. It cannot substitute for the signed-message flow's guarantee.

## Response fields

| Field | Description |
| --- | --- |
| `id` | `bw_...` |
| `name` | The label you supplied |
| `network` | One of the supported networks above |
| `is_account_abstraction` | Whether the wallet was registered by direct address (`true`) or signed message (`false`) |
| `address` | The wallet address, lower-cased for EVM networks |
| `signature_tx_hash` | The signature used to derive `address`, present only when the wallet was registered via the signed-message flow (`is_account_abstraction: false`) |

## List and retrieve wallets

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID, `re_000000000000` with your customer ID.

List every wallet registered for a customer:

```bash [cURL]
curl https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/blockchain-wallets \
  --header 'Authorization: Bearer YOUR_API_KEY'
```

Retrieve a single wallet by ID:

```bash [cURL]
curl https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/blockchain-wallets/bw_000000000000 \
  --header 'Authorization: Bearer YOUR_API_KEY'
```

Both return the [response fields](#response-fields) above. `name` and `network` can come back `null` on a small number of wallets created before BlindPay required those fields; treat them as optional when reading the response.

## Remove a blockchain wallet

```bash [cURL]
curl --request DELETE \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/blockchain-wallets/bw_000000000000 \
  --header 'Authorization: Bearer YOUR_API_KEY'
```

This is a soft delete: the wallet stops appearing in list and retrieve calls and can no longer be used as a payin or payout target, but the record itself is retained. No webhook fires on deletion, only [`blockchainWallet.new`](#webhooks) fires on creation.

**Warning:**

If the wallet is still set as `blockchain_wallet_id` on a [virtual account](../virtual-accounts/virtual-accounts.md) that isn't deleted or rejected, deletion fails with `wallet_attached_to_virtual_account`. Delete the virtual account first.

## Fund a test wallet with USDB

On a development instance you can mint [USDB](../kb/supported-chains.md), BlindPay's test stablecoin, straight into a wallet instead of routing a test payin through it. This only works on `stellar_testnet` and `solana_devnet`; there is no equivalent for EVM testnets or Tron.

**Note:**

This is a devnet/testnet-only tool for exercising payouts and balance checks in development. It has no production equivalent, since USDB has no mainnet deployment.

You need the Stellar or Solana address you want to fund. It does not have to already be registered as a blockchain wallet on BlindPay.

### Stellar: create a trustline, then mint

Stellar requires an account to explicitly opt in to holding an asset before it can receive any. USDB on Stellar also requires issuer authorization, so minting is a two-step process.

### Create the trustline transaction

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/create-asset-trustline \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "address": "YOUR_STELLAR_WALLET_ADDRESS"
  }'
```

The response includes an unsigned `changeTrust` XDR for the USDB testnet asset. Sign it with the wallet's own Stellar key before continuing; you do not need to submit it to the network yourself.

### Mint USDB

Call mint with the signed XDR. BlindPay submits the trustline transaction, waits for it to settle, then mints the amount to the account:

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/mint-usdb-stellar \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "address": "YOUR_STELLAR_WALLET_ADDRESS",
    "amount": "100",
    "signedXdr": "YOUR_SIGNED_XDR"
  }'
```

`signedXdr` is optional: if you already submitted the trustline transaction yourself, omit it and call this endpoint with just `address` and `amount`.

| Error | Meaning |
| --- | --- |
| `blockchain_wallet_address_missing` | `address` was empty. |
| `trustline_not_found` | The account has no USDB trustline yet. Call `create-asset-trustline`, sign it, and retry. |
| `trustline_not_authorized` | The trustline exists but issuer authorization failed. |
| `trustline_full` | The trustline's limit is already reached; it cannot accept more USDB. |
| `asset_issuer_not_found` | The USDB issuer account could not be found. |
| `mint_usdb_failed` | Any other minting failure. |

### Solana: mint directly

Solana needs no trustline step; the associated token account is created automatically.

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/mint-usdb-solana \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "address": "YOUR_SOLANA_WALLET_ADDRESS",
    "amount": "100"
  }'
```

`blockchain_wallet_address_missing` (400) if `address` is empty, `mint_usdb_solana_failed` (400) for any other minting failure.

## Webhooks

| Event | Fires when |
| --- | --- |
| `blockchainWallet.new` | A blockchain wallet is successfully created |

See [Webhooks](../essentials/webhooks.md) for delivery and signature verification.

## Related

- [Managed wallet](../wallets/wallets.md): a BlindPay-custodied alternative that needs no customer signature
- [Payin with blockchain wallet](payin-blockchain-wallet.md): use a blockchain wallet as the payin delivery target
- [Payouts](../payouts/payouts.md): authorize a payout from a blockchain wallet, per network
- [Supported chains](../kb/supported-chains.md): full chain and token matrix
- [Webhooks](../essentials/webhooks.md): event delivery and signature verification
