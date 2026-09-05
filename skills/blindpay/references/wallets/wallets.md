# Managed wallets

Create a BlindPay-managed wallet, check its balance, and use it on payins and payouts.

Source: https://blindpay.com/docs/wallets

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

A managed wallet (`bl_...`) is a balance BlindPay creates and manages for a customer. You interact with it in three ways: create it once, check its balance, and reference it on payins and payouts. The settlement machinery behind it is BlindPay's problem, not yours.

**Warning:**

Managed wallets are in beta.

**Advanced flavor (stablecoin and blockchain mechanics)**

A managed wallet is a BlindPay-custodied wallet: BlindPay generates the address, holds the keys, and your customer's balance moves through your product rather than a browser extension or a signed transaction. This page is the full reference for the managed wallet entity (`bl_...`). For the customer-controlled alternative, see [blockchain wallets](../payins/blockchain-wallets.md).

**Warning:**

Managed wallets are in beta. Only the chains and tokens listed below have confirmed support, don't rely on any network not in the table.

## Supported chains and tokens

| Chain | Development | Production | Tokens |
| --- | --- | --- | --- |
| Ethereum | `sepolia` | `ethereum` | USDC, USDT |
| Base | `base_sepolia` | `base` | USDC, USDT |
| Polygon | `polygon_amoy` | `polygon` | USDC, USDT |
| Arbitrum | `arbitrum_sepolia` | `arbitrum` | USDC, USDT |
| Solana | `solana_devnet` | `solana` | USDC, USDT |

`USDB` is also available on every development network above, it's BlindPay's test stablecoin and only exists on development instances.

**Note:**

Stellar and Tron are not currently supported for managed wallets. Create a [blockchain wallet](../payins/blockchain-wallets.md) instead if you need those chains.

## Prerequisites

**Before you start:**

1. Create an account at https://app.blindpay.com/sign-up
2. Create a development instance (see essentials/instances.md)
3. Create your API key (see essentials/api-keys.md)

A customer must exist and have `kyc_status` of `approved`, `approved_rfi`, or `compliance_request` before you can create a wallet for them. A customer can hold up to 10 wallets.

**Note:**

A customer in `compliance_request` can still create and use a managed wallet, and can still move funds already held in one (transfers, balance checks). Fiat rails stay gated: payin and payout quotes require `approved` or `approved_rfi`, so a `compliance_request` customer can't bring new fiat in or out until the open [RFI](../essentials/rfi.md) is resolved.

**Note:**

Managed wallets are gated by the `wallets_and_transfers` subscription feature on your instance. Every wallet endpoint (create, get, list, balance, delete) returns `400 wallets_and_transfers_not_enabled` if it isn't turned on. Contact BlindPay to enable it.

## Create a managed wallet

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID, `re_000000000000` with your customer ID.

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/wallets \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "network": "base",
    "name": "Customer Balance"
  }'
```

`network` is the settlement rail BlindPay uses under the hood. Use `base` in production and `base_sepolia` on development instances unless you have a reason to pick another; the full list is on the Advanced flavor of this page.

Save two things from the response: the `id` (`bl_...`), which payin quotes reference, and the `address`, which funds payouts.

```json
{
  "id": "bl_000000000000",
  "network": "base",
  "address": "0x1234567890abcdef1234567890abcdef12345678",
  "name": "Customer Balance",
  "created_at": "2026-01-01T00:00:00.000Z"
}
```

**Advanced flavor (stablecoin and blockchain mechanics)**

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID, `re_000000000000` with your customer ID.

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/wallets \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "network": "polygon",
    "external_id": "your-database-id",
    "name": "Blindpay Wallet"
  }'
```

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `network` | string | Yes | One of the chains in the table above |
| `name` | string | Yes | Max 255 characters |
| `external_id` | string | No | Your own correlation ID, max 255 characters |

BlindPay generates the address server-side, you never supply one. Creating a wallet fires a `wallet.new` webhook with the same payload as the GET response. The response returns the wallet, including its `id` (`bl_...`) and `address`:

```json
{
  "id": "bl_000000000000",
  "network": "polygon",
  "address": "0x1234567890abcdef1234567890abcdef12345678",
  "name": "Blindpay Wallet",
  "external_id": "your-database-id",
  "created_at": "2026-01-01T00:00:00.000Z"
}
```

### Get a managed wallet

```bash [cURL]
curl --request GET \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/wallets/bl_000000000000 \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json'
```

**Note:**

Easy to mix up: `bl_` is a managed wallet, `bw_` is a blockchain wallet. The two prefixes don't share an obvious naming split, so double check which one you're passing into a quote or payout.

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

## Check the balance

```bash [cURL]
curl --request GET \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/wallets/bl_000000000000/balance \
  --header 'Authorization: Bearer YOUR_API_KEY'
```

```json
{
  "USDC": { "address": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913", "id": "usdc", "symbol": "USDC", "amount": 100, "earning_amount": 0, "total_amount": 100 },
  "USDT": { "address": null, "id": null, "symbol": "USDT", "amount": 0, "earning_amount": 0, "total_amount": 0 },
  "USDB": { "address": null, "id": null, "symbol": "USDB", "amount": 0, "earning_amount": 0, "total_amount": 0 }
}
```

The response always includes all three tokens; one the wallet doesn't hold comes back with `amount: 0`. `amount` is the idle balance, `earning_amount` is the part earning [yield](https://blindpay.com/docs/yield), and `total_amount` is the sum.

## Receive payments into it

Pass the wallet's `id` as `wallet_id` on a [payin quote](../payins/payin-quotes.md) and the deposit settles into the wallet once the payin completes. Two webhooks confirm it: `payin.complete` and `wallet.inbound`. [Virtual account](../virtual-accounts/virtual-accounts.md) deposits settle into their linked wallet the same way.

## Pay out from it

Pass the wallet's `address` as `sender_wallet_address` when you [execute a payout](../payouts/payouts.md). Because BlindPay manages the wallet, the payout is a single call, no extra authorization step.

## Yield

A wallet can earn on the balance it holds. Turn it on with one call and the idle balance moves into a lending vault; payouts keep working and pull the funds back when needed. `yield_status` on the wallet tells you whether it is on. See [Yield](https://blindpay.com/docs/yield).

## Related

- [Store](store.md): how the wallet fits between payins and payouts
- [Yield](https://blindpay.com/docs/yield): earn on the wallet balance
- [Payins](../payins/payins.md): accept a bank transfer and credit the wallet
- [Payouts](../payouts/payouts.md): pay out from the wallet to a bank account
- [Webhooks](../essentials/webhooks.md): `wallet.inbound` and the payment lifecycle events

**Advanced flavor (stablecoin and blockchain mechanics)**

## Receive stablecoins

Share the wallet `address` with anyone sending stablecoins from an external wallet. There is no approval or signature step on the receiving end, any transfer to that address that arrives on the matching network credits the wallet.

Every time stablecoins land in the wallet, a `wallet.inbound` webhook fires with this payload:

```json [wallet.inbound]
{
  "id": "bl_000000000000",
  "address": "0x1234567890abcdef1234567890abcdef12345678",
  "network": "base",
  "token": {
    "address": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "id": "usdc",
    "symbol": "USDC",
    "amount": 100
  }
}
```

**Warning:**

`wallet.inbound` reports `amount` scaled by 100 (so `100` means $1.00), while the wallet balance endpoint reports the raw amount. Don't assume the two use the same unit.

`wallet.inbound` only fires for USDC and USDT deposits. See [webhooks](../essentials/webhooks.md) for signature verification.

## Collect fiat

You can collect a fiat payment and have the equivalent stablecoins delivered straight into a managed wallet. This is the same on-ramp flow as any payin, with one difference: pass `wallet_id` instead of `blockchain_wallet_id` when creating the payin quote.

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/payin-quotes \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "wallet_id": "bl_000000000000",
    "currency_type": "sender",
    "cover_fees": true,
    "request_amount": 1000,
    "payment_method": "pix",
    "token": "USDC"
  }'
```

Continue with `POST /payins/evm` using the resulting `payin_quote_id`. Stablecoins land in the managed wallet's balance and a `wallet.inbound` webhook fires alongside `payin.complete`. See [Payin with managed wallet](../payins/payin-managed-wallet.md) for the step-by-step flow and [payins](../payins/payins.md) for payment methods and statuses.

## Send fiat

You can fund a payout from a managed wallet's balance instead of an external wallet. Create a payout quote, then pass the managed wallet's address as `sender_wallet_address` when you execute the payout. Because BlindPay already custodies the wallet, there is no `approve` call or signature step, the payout executes directly. [Payout with managed wallet](../payouts/payout-managed-wallet.md) walks through the full flow.

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID, `re_000000000000` with your customer ID, `ba_000000000000` with your bank account ID.

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/payouts/evm \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "quote_id": "qu_000000000000",
    "sender_wallet_address": "0x1234567890abcdef1234567890abcdef12345678"
  }'
```

See [Payout with managed wallet](../payouts/payout-managed-wallet.md) for the step-by-step flow, and [payouts](../payouts/payouts.md) for statuses and testing amounts.

## Send stablecoins

To move stablecoins out of a managed wallet to another managed wallet or an external address, create a transfer quote and execute it before the quote expires. See [Send](send.md) for the full request and response fields.

**Note:**

USDC transfers can move across Ethereum, Polygon, Base, and Arbitrum using Circle CCTP v2; every other token still requires the same network on both sides. The transfer quote expires in 15 seconds, the shortest of any BlindPay quote.

## Yield

A managed wallet can earn on its USDC balance. Enable yield and BlindPay sweeps the idle balance into an ERC-4626 lending vault on the wallet's network, redeems from it when a payout or transfer needs the funds, and reports the position through the yield endpoints and `earning_amount` on the balance. `yield_status` on the wallet is `disabled`, `enabled`, or `disabling`, and every change fires `wallet.update`. See [Yield](https://blindpay.com/docs/yield).

## Related

- [Yield](https://blindpay.com/docs/yield): earn on the wallet balance
- [Blockchain wallets](../payins/blockchain-wallets.md): the customer-controlled alternative to a managed wallet
- [Payins](../payins/payins.md): collect fiat and deliver stablecoins to a wallet
- [Payouts](../payouts/payouts.md): convert a wallet's stablecoin balance to fiat
- [Send](send.md): move stablecoins between wallets
- [Webhooks](../essentials/webhooks.md): `wallet.new`, `wallet.update`, `wallet.inbound`
