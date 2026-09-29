# Migrate from Dynamic to BlindPay

Keep Dynamic for embedded and server wallets, move the stablecoin-to-fiat offramp leg to BlindPay: register the Dynamic wallet, quote and execute payouts, and verify webhooks.

Source: https://blindpay.com/docs/migrations/dynamic

This guide is for teams using Dynamic (docs.dynamic.xyz) as their wallet layer who need a stablecoin-to-fiat offramp. Dynamic stays in place for embedded and server wallets: users' stablecoins keep living where they are. The migration only replaces whatever currently converts those stablecoins to fiat and pays it out, wiring the Dynamic wallet into BlindPay's quote-then-payout flow and moving completion detection onto BlindPay's webhooks.

## How the concepts map

| Dynamic concept | BlindPay concept |
| --- | --- |
| Payee / end user | Customer (`re_`), onboarded through KYC or KYB |
| Payout destination (bank details on file) | Bank account (`ba_`) on the relevant rail |
| Dynamic embedded or server wallet that sends the crypto leg | Registered blockchain wallet (`bw_`), used as `sender_wallet_address` on a payout |
| Funding integration or manual offramp flow | Quote (`qu_`) followed by payout (`po_`) |
| Funding completion webhook or polling | Svix-signed webhooks (`payout.new`, `payout.update`, `payout.complete`) |

Dynamic's funding surface is built for onramp (cards, Apple Pay, Google Pay, exchange transfers), not a native offramp. Confirm in your own codebase which mechanism actually handles the stablecoin-to-fiat leg today (a Dynamic funding integration, a bolted-on third-party offramp, or a manual flow) instead of assuming one. Anything with no direct BlindPay equivalent should go in a written gap list rather than being guessed at.

## Migration steps

### Inventory the current offramp path

List every stored Dynamic wallet ID (embedded or server, and whether it signs directly or through Delegated Access) that sends the crypto leg, any stored provider addresses or transaction hashes, and how completion is detected today. Write down the mapping from what you actually use to BlindPay's [customers](../essentials/customers.md), [bank accounts](../payouts/bank-accounts.md), and [blockchain wallets](../payins/blockchain-wallets.md), and flag anything without a direct equivalent instead of guessing.

### Register wallets and re-run KYC

Register each Dynamic wallet as a BlindPay [blockchain wallet](../payins/blockchain-wallets.md): fetch the sign-message challenge, sign it with the Dynamic wallet's own signing call, submit the signature, and persist the returned `bw_` ID against the customer record. KYC does not transfer between providers, so re-onboard each payee through [customer creation and document upload](../essentials/customers.md), sequencing it ahead of cutover so approved customers are ready when a flow switches over.

### Rebuild money movement as quote then payout

Replace the old offramp call with BlindPay's two-step model: request a [payout quote](../payouts/payout-quotes.md), sign and confirm the approve call through the same Dynamic wallet, then execute the [payout](../payouts/payout-evm.md) with `sender_wallet_address` set to the Dynamic wallet's address, before the quote expires. Store the `qu_`, `po_`, and `bw_` IDs alongside whatever ID the old offramp path used for the same transfer.

### Switch completion detection to webhooks

Replace the old offramp completion signal with BlindPay's [Svix-signed webhooks](../essentials/webhooks.md): [verify](../essentials/webhooks-verification.md) `svix-id`, `svix-timestamp`, and `svix-signature` against the raw request body, dedup on `svix-id`, and update payout state on the [`payout.new`, `payout.update`, and `payout.complete` events](../essentials/webhooks-events.md).

### Cut over per payee or corridor

Roll out behind a feature flag, one payee or corridor at a time. Dual-run new payouts on BlindPay while the old offramp path finishes any in-flight transfers, keep both completion signals live during the window, and only retire the old offramp integration once a full dual-run settlement cycle has completed and reconciled.

## Copy-paste prompt

Paste this into your coding agent to run the migration end to end.

```text [Migration prompt]
You are migrating my application's stablecoin-to-fiat offramp leg from Dynamic (docs.dynamic.xyz) to BlindPay. Dynamic stays in place as the wallet layer: users' stablecoins keep living in their Dynamic embedded or server wallets. This migration only replaces whatever converts those stablecoins to fiat and pays it out.

Before writing code, read these sources and follow them over any prior knowledge:
- https://blindpay.com/docs/llms.txt (read the quickstarts, customers, bank accounts, quotes, blockchain wallets, and webhooks pages)
- The OpenAPI spec: curl https://api.blindpay.com/doc
- https://www.dynamic.xyz/features/funding
- https://www.dynamic.xyz/docs/overview/developer-dashboard/webhooks/overview
Dynamic itself does not settle fiat; its funding surface is built for onramp (cards, Apple Pay, Google Pay, exchange transfers) rather than a native offramp. Confirm in the Dynamic docs which mechanism this codebase actually wired up for the stablecoin-to-fiat leg (a Dynamic funding integration, a bolted-on third-party offramp, or a manual flow) instead of assuming one; if the code shows something not covered by these pages, phrase it as unconfirmed rather than inventing an endpoint.

Do the migration in this order:
1. Inventory the current offramp path from the codebase: the funding or offramp integration in use, every stored Dynamic wallet ID (embedded or server, and whether it signs directly or through Delegated Access) that sends the crypto leg, any stored provider addresses or transaction hashes, and how completion is detected today (webhook, polling, or manual confirmation). Produce a written mapping from what we actually use to BlindPay concepts: the payee becomes a BlindPay customer (re_) with KYC/KYB, their payout destination becomes a bank account (ba_) on the relevant rail, and the Dynamic wallet that currently sends to the offramp destination becomes the sender_wallet_address on a BlindPay payout instead. Flag anything with no direct equivalent rather than guessing.
2. Register each Dynamic wallet as a BlindPay external wallet: fetch the sign-message challenge (GET /v1/instances/{instance_id}/customers/{customer_id}/blockchain-wallets/sign-message), sign it with the Dynamic wallet's own signing call (the embedded wallet's client SDK, or delegatedSignMessage on the server for a Delegated Access wallet, per the Dynamic docs), submit the signature to POST /v1/instances/{instance_id}/customers/{customer_id}/blockchain-wallets, and persist the returned bw_ ID against the customer record. Re-onboard each payee's KYC/KYB on BlindPay through terms of service, customer creation, and document upload driven by webhooks, since KYC does not transfer between providers; sequence this ahead of the cutover so approved customers are ready when a flow switches over.
3. Rebuild money movement on BlindPay's explicit two-step model: request a quote (POST /v1/instances/{instance_id}/quotes), sign and confirm the returned approve call through the same Dynamic wallet, then execute the payout (POST /v1/instances/{instance_id}/payouts/evm) with sender_wallet_address set to the Dynamic wallet's address, before the quote's expires_at (about 5 minutes). Store qu_, po_, and bw_ IDs alongside whatever ID the old offramp path used for the same transfer.
4. Replace the old offramp completion signal with BlindPay's Svix-signed webhooks: verify svix-id, svix-timestamp, and svix-signature against the raw request body with whsec_, dedup on svix-id, and update payout state on payout.new, payout.update, and payout.complete.
5. Cut over per payee or corridor behind a feature flag: dual-run new payouts on BlindPay while the old offramp path finishes any in-flight transfers, keep both completion signals live during the window, and only retire the old offramp integration once a full dual-run settlement cycle has completed and reconciled.

Constraints:
- Develop against a BlindPay development instance (USDB on testnets, KYC auto-approves) first, and exercise the $666.00 forced-failed and $777.00 forced-refunded sentinel amounts before touching production traffic.
- Keep BlindPay and Dynamic API keys server-side.
- Store and compare all amounts as integer minor units; never use floating point for money math.
- Produce a written migration report before changing code: the mapping table, the confirmed old-offramp mechanism, gaps, and the re-KYC plan.

Deliverables: the migration report, the external-wallet registration module (challenge fetch, Dynamic signing call, bw_ persistence), the quote-to-payout orchestration with expiry handling, webhook handlers for the three payout events, and a cutover checklist.
```

## Before you cut over

- Develop and test against a [development instance](../essentials/sandbox-vs-production.md) first, where USDB runs on testnets and KYC auto-approves.
- Exercise the $666.00 forced-failed and $777.00 forced-refunded sentinel amounts before sending production traffic.
- Store and compare all amounts as integer minor units; never use floating point for money math.
- Keep BlindPay and Dynamic API keys server-side.
- Dual-run at least one full settlement cycle per payee or corridor, including the sentinel failure and refund cases, before retiring the old offramp path.

## Related docs

- [Blockchain wallets](../payins/blockchain-wallets.md)
- [Payout quotes](../payouts/payout-quotes.md)
- [Payouts on EVM](../payouts/payout-evm.md)
- [Webhooks](../essentials/webhooks.md)
- [Customers and KYC](../essentials/customers.md)
