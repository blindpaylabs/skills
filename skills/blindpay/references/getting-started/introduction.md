# Introduction

BlindPay is a global payment API that moves money over bank rails and stablecoins from a single integration.

Source: https://blindpay.com/docs/introduction

BlindPay is a global payment infrastructure API. One integration covers both traditional bank transfers and stablecoin payments, so your product can move money across borders without stitching together separate rails.

Under the hood, every payment settles through stablecoins. BlindPay is a non-custodial payment processor: your funds stay under your control throughout the process, and if a transaction fails, funds are automatically returned to the originating wallet. You don't need to think about that layer to use the fiat side of the API, but it's why BlindPay can move money between bank rails and blockchains in the first place.

## What BlindPay does

The API is organized around four verbs that work the same way whether you think in fiat or in stablecoins.

| Verb | Abstracted | Advanced |
| --- | --- | --- |
| **Store** | N/A (value settles as stablecoin in the linked wallet) | Managed wallet (or external blockchain wallet) |
| **Receive** | Bank transfer in (payin) | On-ramp: fiat in, stablecoin delivered to a wallet |
| **Send** | Pay out to a bank account (payout) | Off-ramp: stablecoin pulled from a wallet, fiat out |
| **Transfer** | N/A | Cross-chain stablecoin transfer |

## Two flavors, one API

The docs are split into two flavors based on how you think about money movement. Both run on the same engine, same authentication, same instances, and same webhooks; pick the lens that matches your product.

- **[Abstracted](https://blindpay.com/docs/overview?flavor=abstracted)** is for builders who think in familiar payment concepts: money in over ACH, wire, SWIFT, or Pix, and money out to a bank account. A virtual account is a customer's dedicated deposit account, payins are deposits, and payouts are bank transfers. Stablecoins settle in the linked wallet behind the scenes.
- **[Advanced](https://blindpay.com/docs/overview?flavor=advanced)** exposes the full stablecoin and blockchain stack. It covers managed and external blockchain wallets, chains and tokens, on-chain authorization (ERC-20 approve, Stellar signing, Solana delegation), and cross-chain transfers.

You can switch flavors at any time from the sidebar; nothing about your account or API key changes.

## How it works

Every payment flows through a customer that has completed KYC. BlindPay handles the compliance layer: you collect the data, BlindPay verifies it. Once a customer is approved, every payment follows the same three-step pattern:

### Quote

Create a quote for the amount you want to move. The quote locks in the exchange rate and fees for a short window (every quote expires in 5 minutes, except an OTC payin quote, which expires in 10 seconds).

### Authorize

Depending on the flow, this means approving a bank account, signing a wallet authorization, or simply holding a valid quote, no extra step needed for most fiat payins.

### Execute

Submit the payment against the quote before it expires. BlindPay moves the funds and reports status changes over webhooks.

**Note:**

BlindPay is non-custodial: your funds remain under your control throughout the process. If a payment can't settle, funds are returned to the originating wallet or bank account.

## Get started

### Pick your flavor

Read the [Abstracted](https://blindpay.com/docs/overview?flavor=abstracted) or [Advanced](https://blindpay.com/docs/overview?flavor=advanced) index to see which lens matches what you're building. Most teams only need one.

### Create an instance

An [instance](../essentials/instances.md) is your sandboxed or production environment, with its own API keys, customers, and webhooks.

### Follow a quickstart

[Bank transfer to stablecoins](quickstart-payin.md), or [stablecoins to bank transfer](quickstart-payout.md).

## Related

- [Abstracted](https://blindpay.com/docs/overview?flavor=abstracted): bank rails, virtual accounts, payins and payouts
- [Advanced](https://blindpay.com/docs/overview?flavor=advanced): wallets, chains, on-ramp and off-ramp
- [Instances](../essentials/instances.md): sandbox vs production environments
- [Build with AI](build-with-ai.md): use BlindPay from an AI coding assistant
