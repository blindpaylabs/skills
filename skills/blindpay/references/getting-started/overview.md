# Overview

Connect to bank rails or stablecoin networks (or both) through one REST API. Issue virtual accounts and move fiat payins and payouts, or hold, send, and receive USDC and USDT across chains.

Source: https://blindpay.com/docs/overview

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

BlindPay's fiat APIs connect your product to bank rails in the US, Brazil, Mexico, Argentina, Colombia, and Europe. Issue a virtual account in each customer's name, accept deposits over ACH, wire, or Pix, and pay out to any bank account (including SWIFT), all through a single REST API.

Under the hood, every dollar or real you move settles through stablecoins. You don't need to think about that layer to use the fiat APIs: BlindPay manages the conversion step for you and hands you bank-rail primitives (virtual accounts, payins, payouts) on top. If you want the crypto mechanics, switch to the Advanced flavor of this page.

**Note:**

BlindPay is non-custodial under the hood: funds always move through a customer that has completed KYC, and a quote (which locks the rate and fees) before any transfer executes.

## What's available

| Capability | Description |
| --- | --- |
| Virtual accounts | A dedicated US bank account issued in each customer's name, with its own routing and account number |
| Receive (payins) | Accept ACH, wire, RTP, SWIFT, Pix, SPEI, Transfers, or PSE deposits; every incoming payment is tracked as a payin |
| Send (payouts) | Pay out to any bank account: ACH, wire, SWIFT, RTP, Pix, SPEI, ACH COP, Transfers, or SEPA |
| Webhooks | Real-time events for every deposit and payout state change |

## Supported corridors

| Payment method | Currency | Direction | Settlement |
| --- | --- | --- | --- |
| ACH | USD | Receive | Up to 5 business days |
| ACH | USD | Send | ~2 business days |
| Wire | USD | Receive | Up to 5 business days |
| Wire | USD | Send | ~1 business day |
| SWIFT (international) | USD | Receive | Up to 5 business days |
| SWIFT (international) | USD | Send | ~5 business days |
| RTP | USD | Receive | Instant |
| RTP | USD | Send | Instant |
| Pix | BRL | Receive + send | Instant (up to 5 min to confirm) |
| SPEI | MXN | Receive + send | Instant (up to 10 min to confirm) |
| Transfers | ARS | Receive + send | Instant (up to 10 min to confirm) |
| PSE | COP | Receive | Up to 10 min (payment link) |
| ACH COP | COP | Send | ~1 business day |
| SEPA | EUR | Send | ~1 business day |

Receive-side timing applies to production instances. On development instances, every payin auto-completes about 30 seconds after initiation, regardless of method.

## The fiat model

The same model applies, read through a bank-rails lens:

| Verb | What it means in fiat terms | Docs |
| --- | --- | --- |
| Receive | A bank deposit (ACH, wire, Pix, SPEI, Transfers, PSE) arrives and is recorded as a payin | [Bank transfer in](../payins/payins.md) |
| Send | A payout moves funds out to any bank account you've added for a customer | [Pay out to bank](../payouts/payouts.md) |

## How funds move

Every payin and payout still settles through stablecoins behind the scenes. You don't sign anything on-chain to use the fiat APIs (BlindPay custodies the conversion step); these diagrams show what happens after you call the API.

### Fiat in (payin)

![Fiat in: payin flow of funds from a sender's bank account to BlindPay](https://pub-4fabf5dd55154f19a0384b16f2b816d9.r2.dev/blindpay-payins-flow-of-funds.png)

1. **Deposit.** The sender moves funds from their bank account to either a virtual account issued for that customer, or BlindPay's own bank details with a memo code.
2. **Confirmation and delivery.** Once the deposit is confirmed, BlindPay completes the payin and the equivalent value lands wherever you configured it to (a managed wallet balance, ultimately payable out again, or an external wallet if you're integrating with the stablecoin side).

### Fiat out (payout)

![Fiat out: payout flow of funds from BlindPay to a recipient's bank account](https://pub-4fabf5dd55154f19a0384b16f2b816d9.r2.dev/blindpay-payouts-flow-of-funds.png)

1. **Authorization.** Funds are authorized for the payout amount (handled for you when paying out of a BlindPay-managed wallet balance; see [Pay out to bank](../payouts/payouts.md)).
2. **Collection and conversion.** BlindPay collects the authorized funds and starts the fiat transfer over the local rail: ACH or wire in the US, Pix in Brazil, SPEI in Mexico, Transfers in Argentina, and so on.
3. **Settlement.** Once the receiving bank confirms the transfer, the payout completes and the recipient has their funds in local currency.

**Warning:**

If the fiat transfer can't settle or gets returned, funds are sent back automatically. Nothing sits in limbo.

## Get started

- [Bank transfer to stablecoins](quickstart-payin.md): create a customer and accept a deposit, step by step
- [Virtual accounts](../virtual-accounts/virtual-accounts.md): how virtual accounts work, approval stages, and required fields
- [Bank transfer in](../payins/payins.md): payin quotes, payment methods, and what to show the payer
- [Pay out to bank](../payouts/payouts.md): add a bank account and send a payout

## Related

- [Introduction](introduction.md)
- [Learn: instances](../essentials/instances.md)
- [Learn: webhooks](../essentials/webhooks.md)
- [Knowledge base: KYC](../kb/kyc.md)
- Switch to the Advanced flavor of this page for the stablecoin mechanics

**Advanced flavor (stablecoin and blockchain mechanics)**

BlindPay's stablecoin APIs connect your product to multiple blockchains and fiat payment rails from a single REST API. Hold USDC and USDT in a wallet, accept fiat and deliver stablecoins on-ramp, pull stablecoins and pay out fiat off-ramp, or move stablecoins cross-chain.

**Warning:**

BlindPay does not provide wallet services. We operate as a non-custodial payment processor: your funds stay under your control throughout the flow. If a transaction fails, funds are automatically returned to the originating wallet.

## What's available

| Capability | Description |
| --- | --- |
| Managed wallets | BlindPay-custodied wallets that hold USDC/USDT per customer (beta, Arbitrum + Polygon) |
| External blockchain wallets | Customer-controlled wallets you register to receive or authorize stablecoin movement |
| Payins (on-ramp) | Accept a fiat deposit and deliver stablecoins to any wallet |
| Payouts (off-ramp) | Pull stablecoins from a wallet and pay out fiat to a bank account |
| Send / Receive | Move stablecoins in and out of managed wallets (USDC can move cross-chain via Circle CCTP v2, other tokens stay same-network) |
| Webhooks | Real-time events for every wallet deposit, payout, and transfer |

## Managed wallets vs external blockchain wallets

BlindPay supports two ways to hold the stablecoin side of a customer's funds. Pick one per customer, or use both.

| | Managed wallet (`bl_...`) | External blockchain wallet (`bw_...`) |
| --- | --- | --- |
| Custody | BlindPay-custodied | Customer-controlled (EOA or account abstraction) |
| Created via | `POST /customers/{re}/wallets` | `POST /customers/{re}/blockchain-wallets` |
| Networks | Arbitrum, Polygon (beta) | Any supported chain |
| Signing required | None, BlindPay moves funds on your behalf | ERC-20 `approve`, Stellar signed XDR, or Solana token delegation depending on chain |
| Typical use | Hold a balance inside BlindPay between on-ramp and off-ramp | Receive funds into a wallet you already control, or fund a payout from your own wallet |
| Add securely | n/a | Sign a message (no manual address entry) |
| Add non-secure | n/a | Paste the address directly (funds are lost if the address is wrong) |

A customer must exist and complete KYC before you can create either kind of wallet. See [Payins](../payins/payins.md) and [Payouts](../payouts/payouts.md) for how each wallet type plugs into a payin or payout.

## Supported chains and tokens

| Chain | Mainnet | Testnet | Tokens |
| --- | --- | --- | --- |
| Ethereum | `ethereum` | `sepolia` | USDC, USDT |
| Base | `base` | `base_sepolia` | USDC, USDT |
| Polygon | `polygon` | `polygon_amoy` | USDC, USDT |
| Arbitrum | `arbitrum` | `arbitrum_sepolia` | USDC, USDT |
| Stellar | `stellar` | `stellar` (testnet) | USDC |
| Solana | `solana` | `solana_devnet` | USDC, USDT |
| Tron (beta) | `tron` | n/a | USDT only |

On development instances, every chain above also accepts **USDB**: a BlindPay test ERC-20/asset used to simulate transactions without real funds. Fund a wallet with it by creating a [payin](../payins/payins.md), which auto-completes on development instances.

Authorization mechanics differ per network when moving stablecoins out of an external wallet:

| Network type | Authorization |
| --- | --- |
| EVM (Ethereum, Base, Polygon, Arbitrum) | Standard ERC-20 `approve` |
| Stellar | Authorize endpoint, then a signed XDR transaction |
| Solana | Token delegation: prepare, sign, submit |

## The verbs, in crypto terms

| Verb | What happens | Docs |
| --- | --- | --- |
| Store | Hold USDC/USDT in a BlindPay-managed wallet | [Store](../wallets/store.md) |
| Send | Stablecoins move out of a managed wallet to another wallet or address | [Send](../wallets/send.md) |
| Receive | Stablecoins arrive in a managed wallet, tracked via `wallet.inbound` | [Receive](../wallets/receive.md) |
| Payin | On-ramp: a fiat deposit is converted and the stablecoin is delivered to a wallet | [Payins](../payins/payins.md) |
| Payout | Off-ramp: stablecoins are pulled from a wallet and paid out as fiat to a bank account | [Payouts](../payouts/payouts.md) |

## Non-custodial flow of funds

BlindPay never takes custody of your stablecoins beyond the single transaction it is asked to execute. Off-ramp (send) and on-ramp (receive) each move through three steps.

### Off-ramp: stablecoin to fiat

![BlindPay payout flow of funds, from wallet authorization to bank settlement](https://pub-4fabf5dd55154f19a0384b16f2b816d9.r2.dev/blindpay-payouts-flow-of-funds.png)

1. **Token authorization**: your wallet authorizes BlindPay to move the exact stablecoin amount needed (ERC-20 `approve` on EVM, a signed XDR on Stellar, or token delegation on Solana).
2. **Token collection and fiat conversion**: BlindPay collects the authorized stablecoins and initiates the matching fiat transfer over the local banking rail (ACH/wire in the US, Pix in Brazil, and other region-specific rails).
3. **Settlement confirmation**: once the receiving bank confirms the fiat transfer, the payout is finalized and the recipient receives funds in their local currency.

**Note:**

If the fiat transfer can't settle, or it gets returned, the stablecoins are sent back to the same wallet that authorized the payout.

### On-ramp: fiat to stablecoin

![BlindPay payin flow of funds, from fiat deposit to stablecoin delivery](https://pub-4fabf5dd55154f19a0384b16f2b816d9.r2.dev/blindpay-payins-flow-of-funds.png)

1. **Fiat deposit**: funds move from a sender's bank account to a BlindPay bank account or a dedicated virtual account.
2. **Stablecoin delivery**: once the deposit is confirmed, BlindPay mints or transfers the equivalent stablecoin amount to the destination wallet, a managed wallet (`wallet_id`) or an external blockchain wallet (`blockchain_wallet_id`), at the rate locked in the quote.

## Get started

- [Stablecoins to bank transfer](quickstart-payout.md): the step-by-step payout quickstart
- [Store](../wallets/store.md): hold stablecoins per customer in a managed wallet
- [Payins](../payins/payins.md): accept fiat deposits and deliver stablecoins
- [Payouts](../payouts/payouts.md): pull stablecoins and pay out fiat
- [Send](../wallets/send.md): move stablecoins between wallets

## Related

- [Introduction](introduction.md)
- [Supported chains](../kb/supported-chains.md)
- [Sandbox vs production](../essentials/sandbox-vs-production.md)
- [Webhooks](../essentials/webhooks.md)
