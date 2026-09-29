# Migrate from Privy to BlindPay

Keep Privy for embedded and server wallets, move the stablecoin-to-fiat offramp leg to BlindPay: register the wallet, quote and execute payouts, and verify webhooks.

Source: https://blindpay.com/docs/migrations/privy

This guide is for teams using Privy (docs.privy.io) as their wallet layer who currently hand off the stablecoin-to-fiat leg to a third-party offramp provider, a different funding integration, or a manual flow. Privy stays in place: users keep their stablecoins in Privy embedded or server wallets. The migration only replaces whatever converts those stablecoins to fiat and pays it out, moving that leg onto BlindPay's customer, wallet, quote, and payout objects.

## How the concepts map

| Privy or offramp concept | BlindPay concept |
| --- | --- |
| Payee receiving the payout | Customer (`re_`) with KYC/KYB |
| Payee's payout destination | Bank account (`ba_`) on the relevant rail |
| Privy wallet sending to the offramp provider's address | `sender_wallet_address` on a BlindPay payout |
| Registering that wallet with BlindPay | Blockchain wallet (`bw_`), created from a signed challenge |
| Old offramp price or conversion step | Quote (`qu_`) |
| Old offramp transfer execution | Payout (`po_`) |
| Old offramp completion signal (webhook, polling, manual) | BlindPay webhook events: `payout.new`, `payout.update`, `payout.complete` |

KYC and KYB do not transfer between providers: every payee needs to be re-onboarded on BlindPay, even if they were already verified with the old offramp provider. Treat anything in the current integration that doesn't map cleanly to this table, such as a custom or manual offramp flow, as unconfirmed until you've traced it in the codebase rather than assuming it matches one of Privy's listed funding partners.

## Migration steps

### Inventory the current offramp path

Trace the codebase for the funding or offramp provider integration in use, every stored Privy wallet ID (embedded or server) that sends the crypto leg, any stored provider addresses or transaction hashes, and how completion is detected today. Write down the mapping from what the app actually uses to the BlindPay concepts above, and flag anything with no direct equivalent instead of guessing at one.

### Register Privy wallets and re-onboard KYC

For each Privy wallet, fetch the sign-message challenge, sign it with the wallet's own signing call, and submit the signature to create a [blockchain wallet](../payins/blockchain-wallets.md) record. Re-run KYC/KYB for each payee on BlindPay, driven by [customer creation](../essentials/customers.md), [terms of service](../essentials/terms-of-service.md), and [document upload](../essentials/upload.md) webhooks, and sequence this ahead of cutover so approved customers are ready when a flow switches over.

### Rebuild money movement on the quote-then-execute model

Replace the old offramp conversion and transfer with BlindPay's two-step flow: request a [payout quote](../payouts/payout-quotes.md), sign and confirm the returned approval through the same Privy wallet, then execute the [payout](../payouts/payout-evm.md) with `sender_wallet_address` set to that wallet's address, before the quote expires. Store the `qu_`, `po_`, and `bw_` IDs alongside whatever ID the old offramp provider used for the same transfer.

### Wire up webhooks

Replace the old offramp completion signal with BlindPay's [Svix-signed webhooks](../essentials/webhooks-verification.md): verify `svix-id`, `svix-timestamp`, and `svix-signature` against the raw request body, dedup on `svix-id`, and update payout state from the [payout events](../essentials/webhooks-events.md).

### Cut over per payee or corridor

Roll out behind a feature flag, one payee or corridor at a time. Dual-run new payouts on BlindPay while the old offramp path finishes any in-flight transfers, keep both completion signals live during the window, and only retire the old integration once a full dual-run settlement cycle has completed and reconciled.

## Copy-paste prompt

Paste this into your coding agent to run the migration end to end.

```text [Migration prompt]
You are migrating my application's stablecoin-to-fiat offramp leg from Privy (docs.privy.io) to BlindPay. Privy stays in place as the wallet layer: users' stablecoins keep living in their Privy embedded or server wallets. This migration only replaces whatever converts those stablecoins to fiat and pays it out.

Before writing code, read these sources and follow them over any prior knowledge:
- https://blindpay.com/docs/llms.txt (read the quickstarts, customers, bank accounts, quotes, blockchain wallets, and webhooks pages)
- The OpenAPI spec: curl https://api.blindpay.com/doc
- https://docs.privy.io/wallets/funding/overview
- https://docs.privy.io/recipes/off-ramp-guide
Privy itself does not settle fiat; it hands off a signed onchain transfer to a third-party offramp provider. Confirm in the Privy docs which mechanism this codebase actually wired up (one of those providers, a different funding integration, or a manual flow) instead of assuming; if the code shows something not covered by these pages, phrase it as unconfirmed rather than inventing an endpoint.

Do the migration in this order:
1. Inventory the current offramp path from the codebase: the funding or offramp provider integration in use, every stored Privy wallet ID (embedded or server) that sends the crypto leg, any stored provider addresses or transaction hashes, and how completion is detected today (webhook, polling, or manual confirmation). Produce a written mapping from what we actually use to BlindPay concepts: the payee becomes a BlindPay customer (re_) with KYC/KYB, their payout destination becomes a bank account (ba_) on the relevant rail, and the Privy wallet that currently sends to the offramp provider's address becomes the sender_wallet_address on a BlindPay payout instead. Flag anything with no direct equivalent rather than guessing.
2. Register each Privy wallet as a BlindPay external wallet: fetch the sign-message challenge (GET /v1/instances/{instance_id}/customers/{customer_id}/blockchain-wallets/sign-message), sign it with the Privy wallet's own signing call (the embedded wallet's provider or the server wallet RPC endpoint, per the Privy docs), submit the signature to POST /v1/instances/{instance_id}/customers/{customer_id}/blockchain-wallets, and persist the returned bw_ ID against the customer record. Re-onboard each payee's KYC/KYB on BlindPay through terms of service, customer creation, and document upload driven by webhooks, since KYC does not transfer between providers; sequence this ahead of the cutover so approved customers are ready when a flow switches over.
3. Rebuild money movement on BlindPay's explicit two-step model: request a quote (POST /v1/instances/{instance_id}/quotes), sign and confirm the returned approve call through the same Privy wallet, then execute the payout (POST /v1/instances/{instance_id}/payouts/evm) with sender_wallet_address set to the Privy wallet's address, before the quote's expires_at (about 5 minutes). Store qu_, po_, and bw_ IDs alongside whatever ID the old offramp provider used for the same transfer.
4. Replace the old offramp completion signal with BlindPay's Svix-signed webhooks: verify svix-id, svix-timestamp, and svix-signature against the raw request body with whsec_, dedup on svix-id, and update payout state on payout.new, payout.update, and payout.complete.
5. Cut over per payee or corridor behind a feature flag: dual-run new payouts on BlindPay while the old offramp path finishes any in-flight transfers, keep both completion signals live during the window, and only retire the old offramp integration once a full dual-run settlement cycle has completed and reconciled.

Constraints:
- Develop against a BlindPay development instance (USDB on testnets, KYC auto-approves) first, and exercise the $666.00 forced-failed and $777.00 forced-refunded sentinel amounts before touching production traffic.
- Keep BlindPay and Privy API keys server-side.
- Store and compare all amounts as integer minor units; never use floating point for money math.
- Produce a written migration report before changing code: the mapping table, the confirmed old-offramp mechanism, gaps, and the re-KYC plan.

Deliverables: the migration report, the external-wallet registration module (challenge fetch, Privy signing call, bw_ persistence), the quote-to-payout orchestration with expiry handling, webhook handlers for the three payout events, and a cutover checklist.
```

## Before you cut over

- Develop against a BlindPay development instance first (USDB on testnets, KYC auto-approves) before touching production traffic.
- Exercise the $666.00 forced-failed and $777.00 forced-refunded sentinel amounts to confirm your webhook handlers behave correctly.
- Keep BlindPay and Privy API keys server-side, never in client code.
- Store and compare all amounts as integer minor units, never floating point.
- Dual-run per payee or corridor: keep both completion signals live until a full settlement cycle has reconciled, then retire the old offramp integration.

## Related docs

- [Blockchain wallets](../payins/blockchain-wallets.md)
- [Payout quotes](../payouts/payout-quotes.md)
- [Payout: EVM](../payouts/payout-evm.md)
- [Webhooks events](../essentials/webhooks-events.md)
- [Webhooks verification](../essentials/webhooks-verification.md)
