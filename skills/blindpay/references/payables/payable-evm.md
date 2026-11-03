# Payable with EVM

Register a bill your customer owes, then pay it from an external EVM wallet by quoting it, approving the ERC-20 pull, and executing the payout.

Source: https://blindpay.com/docs/payable-evm

This tutorial registers a [payable](payables.md) and pays it from an external EVM wallet on Ethereum, Base, Polygon, or Arbitrum. All stablecoins on EVM chains are ERC-20 tokens, so authorization means calling `approve` on the token contract to let BlindPay pull the quoted amount.

If the funds live in a BlindPay-custodied wallet instead, skip the approval entirely: see [Payable with managed wallet](payable-managed-wallet.md).

**Note:**

The examples below use `base_sepolia` and `USDB` since they're the development network and test token. Swap in the matching production network and token (`USDC` or `USDT`) when you go live.

Payables are **EVM only** today; a non-EVM network is refused at quote time with `payable_network_not_supported`.

## Prerequisites

**Before you start:**

1. Create an account at https://app.blindpay.com/sign-up
2. Create a development instance (see essentials/instances.md)
3. Create your API key (see essentials/api-keys.md)

You also need:

1. A [customer](../essentials/customers.md) with `kyc_status: "approved"`. There is no bank account to register: the bill names its own beneficiary
2. A real boleto linha digitável or PIX copia e cola code to pay
3. [An RPC provider URL for Base Sepolia](https://chainlist.org/?search=base&testnets=true)
4. [A wallet funded with testnet ether](https://www.alchemy.com/faucets/base-sepolia) to pay gas
5. [The private key, or any way to instantiate the wallet with ethers.js](https://docs.ethers.org/v6/api/wallet/#BaseWallet_new)

### Register the payable

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID, `re_000000000000` with your customer ID.

```js [index.js]
const payableResponse = await fetch(
  'https://api.blindpay.com/v1/instances/in_000000000000/payables',
  {
    method: 'POST',
    headers: {
      'Authorization': 'Bearer YOUR_API_KEY',
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      customer_id: 're_000000000000',
      currency: 'BRL',
      boleto_barcode: '34191790010104351004791020150008191070026000',
    }),
  }
)

const payable = await payableResponse.json()
```

The payable is created in `draft`. Registering the same code twice while a live payable exists fails with `duplicate_payable`; to retry a failed attempt, quote the same payable again (it returns to `draft`), and a deleted draft frees the code for re-registration.

### Quote the payable and approve the tokens

Create a quote with `payable_id` instead of `bank_account_id`. Do not send `request_amount` or `currency_type`: the amount comes from the payable, re-resolved from the rail (fines, interest) for a boleto.

The `contract` object in the response carries everything the `approve` call needs, including `amount` already adjusted for the token's decimals.

```js [index.js]
import { ethers } from 'ethers'

const quoteResponse = await fetch(
  'https://api.blindpay.com/v1/instances/in_000000000000/quotes',
  {
    method: 'POST',
    headers: {
      'Authorization': 'Bearer YOUR_API_KEY',
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      payable_id: payable.id,
      network: 'base_sepolia',
      token: 'USDB',
    }),
  }
)

const quote = await quoteResponse.json()

// What the bill actually costs, resolved at quote time. Show this before
// asking anyone to sign anything.
console.log(quote.receiver_amount, quote.sender_amount)

const provider = new ethers.JsonRpcProvider('YOUR_RPC_URL')
const wallet = new ethers.Wallet('YOUR_PRIVATE_KEY', provider)

const token = new ethers.Contract(
  quote.contract.address,
  quote.contract.abi,
  wallet
)

const approval = await token.approve(
  quote.contract.blindpayContractAddress,
  quote.contract.amount
)
await approval.wait()
```

**Warning:**

The quote expires 5 minutes after creation. If the approval transaction is slow to mine, quote again rather than committing an expired one.

### Execute the payout

Executing the payout is what actually pays the bill. `sender_wallet_address` is the wallet that just approved.

```js [index.js]
const response = await fetch(
  'https://api.blindpay.com/v1/instances/in_000000000000/payouts/evm',
  {
    method: 'POST',
    headers: {
      'Authorization': 'Bearer YOUR_API_KEY',
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      quote_id: quote.id,
      sender_wallet_address: wallet.address,
    }),
  }
)

const payout = await response.json()
```

The payout starts `processing`, and the payable's own `status` now mirrors it. The allowance is not checked when the payout is created: if the wallet has not approved enough, the payout is still returned and then fails during collection with `erc20_allowance_insufficient`. Approve at least `contract.amount` before you execute.

### Track it to settlement

Stablecoins are collected first, then the bill is paid. A PIX code usually completes in under a minute; a boleto settles on a banking day.

Subscribe to `payable.complete` to hear when the bill is paid and `payable.update` for bill-level changes (a payout claimed it, the attempt failed or was refunded and it is payable again, or a draft was deleted). The executing payout emits its own `payout.*` events for the mid-flight detail. See [Payables](payables.md#webhooks) for details.

```js [index.js]
const detail = await fetch(
  `https://api.blindpay.com/v1/instances/in_000000000000/payables/${payable.id}`,
  { headers: { Authorization: 'Bearer YOUR_API_KEY' } }
).then(r => r.json())

console.log(detail.status, detail.amount)
```

## Related

- [Payables](payables.md): registration, amount semantics, lifecycle, and webhooks
- [Payable with managed wallet](payable-managed-wallet.md): the same flow with no approval step
- [Payin with blockchain wallet](../payins/payin-blockchain-wallet.md): fund your test wallet on development instances
