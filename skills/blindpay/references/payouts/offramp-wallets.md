# Offramp wallets

What a BlindPay offramp wallet is, the chains and stablecoins it supports, minimums, and how deposits automatically convert to a fiat payout.

Source: https://blindpay.com/docs/offramp-wallets

An offramp wallet is a blockchain wallet that BlindPay creates and manages for you, tied to one of your customer's bank accounts. Every USDC or USDT deposit it receives is automatically converted to fiat and paid out to that bank account, with no quote or execute call on your side. It is the stablecoin-side mirror of a [virtual account](../virtual-accounts/virtual-accounts.md): where a virtual account turns incoming fiat into stablecoin, an offramp wallet turns incoming stablecoin into fiat.

## How it works

BlindPay always creates a [payout](payouts.md) automatically whenever the wallet receives a USDC or USDT transaction. You share the wallet's deposit address with whoever is paying your customer; the moment funds land, conversion and settlement to the linked bank account happen on their own.

### Supported chains and stablecoins

| Chain | Stablecoins | Minimum | Additional fee |
| --- | --- | --- | --- |
| Tron | USDT only | 200 USDT | 15 USDT |
| Solana | USDC only | 50 USDC | 0.50 USDC |
| Ethereum | USDC only | No fixed minimum* | 1 USDC |
| Polygon | USDC and USDT | No fixed minimum* | 0 USDC |
| Arbitrum | USDC only | No fixed minimum* | 0 USDC |
| Base | USDC only | No fixed minimum* | 0 USDC |
| Tempo | USDC only | No fixed minimum* | 0 USDC |
| Arc | USDC only | No fixed minimum* | 0 USDC |

*The deposit only needs to cover its fees (additional fee, percentage fee, and bank transfer fee). A deposit smaller than the total fees is not converted: no payout or quote is created, and the funds stay at the wallet's deposit address rather than being refunded automatically. Nothing fails an API call here since no API call triggered the deposit, and you won't get a payout webhook for it, so treat deposits below the minimum as something to monitor for and recover manually, not something the API surfaces an error for.

Each mainnet network above has a matching testnet for development instances: `solana_devnet`, `sepolia`, `polygon_amoy`, `arbitrum_sepolia`, `base_sepolia`, `tempo_testnet`, `arc_testnet`. Development instances receive USDB, BlindPay's test stablecoin, instead of USDC/USDT, with no additional fee.

A production instance can only create offramp wallets on mainnet networks; a development instance can only create them on the matching testnet. Requesting the wrong kind fails with `BLOCKCHAIN_NETWORK_NOT_SUPPORTED`.

**Warning:**

If the destination bank account is `type: "sepa"`, only a subset of these networks is available as a deposit network, see [SEPA bank accounts](#sepa-bank-accounts) below.

### Fee example

Sending 100 USDT to an offramp wallet on Tron that settles over ACH. USDT deposits always convert at a flat 1:1 rate with no market spread, so the $1.00 in this example is guaranteed, not assumed. USDC deposits use the live market rate instead.

1. Convert: 100 USDT = $100.00
2. Additional fee: 15 USDT = $15.00
3. Percentage fee: 0.1% of $100.00 = $0.10
4. Bank transfer fee: $0.40 (ACH)

Final: $100.00 - $15.00 - $0.10 - $0.40 = **$84.50** delivered to the recipient's bank account. The additional fee is fixed per chain (see the table above); the percentage fee and bank transfer fee depend on your pricing and the payout rail.

## Create an offramp wallet

An offramp wallet is created on an existing bank account, which becomes its payout destination. Once created, it has a deposit address on the network you chose; anything sent to that address is converted and paid out automatically.

### Prerequisites

**Before you start:**

1. Create an account at https://app.blindpay.com/sign-up
2. Create a development instance (see essentials/instances.md)
3. Create your API key (see essentials/api-keys.md)

You must also [create a customer](../essentials/customers.md) and add a [bank account](bank-accounts.md) before creating an offramp wallet. The wallet inherits that bank account as its payout destination. The bank account can't belong to a [payable](../payables/payables.md): one created for a payable is locked to it, and creating an offramp wallet on it fails with `bank_account_owned_by_payable`.

A bank account can only have one offramp wallet per network. Creating a second one for the same bank account and network fails with `offramp_wallet_already_exists`; save the `id` and `address` from the first response, or use a different network.

### The request

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID, `re_000000000000` with your customer ID.

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/bank-accounts/ba_000000000000/offramp-wallets \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
  "network": "tron"
}'
```

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `network` | enum | Yes | Production: `tron`, `solana`, `ethereum`, `polygon`, `arbitrum`, `base`, `tempo`, `arc`. Development instances use the matching testnets: `solana_devnet`, `sepolia`, `polygon_amoy`, `arbitrum_sepolia`, `base_sepolia`, `tempo_testnet`, `arc_testnet`. |
| `external_id` | string | No | Your own reference for the wallet, echoed back on the response. |

The response includes `id` (`ow_...`), `network`, `address`, `circle_wallet_id` (null for Tempo-network wallets, which use a different custody provider), and your `external_id`. Share `address` with whoever is paying your customer: any USDC or USDT sent to it on the chosen network is converted and paid out to the linked bank account automatically.

If your instance has email notifications enabled, BlindPay emails the customer as soon as the deposit triggers a payout, with the subject `Payout Initiated - <Network> Deposit Received`.

### SEPA bank accounts

An offramp wallet on a `sepa` bank account only supports a subset of deposit networks, since each network needs individual Travel Rule approval. Supported: `ethereum`, `base`, `arbitrum`, `polygon` (and their development testnets), and `solana`/`solana_devnet`. `tron`, `tempo`/`tempo_testnet` and `arc`/`arc_testnet` are not supported for SEPA. Requesting an unsupported network fails with `sepa_offramp_wallet_network_not_supported`.

## Retrieve an offramp wallet

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID, `re_000000000000` with your customer ID.

```bash [cURL]
curl --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/bank-accounts/ba_000000000000/offramp-wallets/ow_000000000000 \
  --header 'Authorization: Bearer YOUR_API_KEY'
```

| Field | Type | Notes |
| --- | --- | --- |
| `id` | string | `ow_...` |
| `external_id` | string or null | Your own reference, if you set one. |
| `instance_id`, `customer_id`, `bank_account_id` | string | The parents this wallet belongs to. |
| `circle_wallet_id` | string or null | Set for Circle-backed networks, null for `tempo`/`tempo_testnet`. |
| `network` | enum | See supported networks above. |
| `address` | string | The deposit address. |
| `created_at`, `updated_at` | date-time | |

There is currently no way to list all of a bank account's offramp wallets through the API. Save the `id` you get back when you create one, and use [Retrieve an offramp wallet](#retrieve-an-offramp-wallet) to look it up later.

## Related

- [Payouts](payouts.md): the payout created automatically on each deposit
- [Bank accounts](bank-accounts.md): the settlement destination
- [Payables](../payables/payables.md): a bank account owned by a payable can't be used for an offramp wallet
- [Virtual accounts](../virtual-accounts/virtual-accounts.md): the fiat-side mirror
