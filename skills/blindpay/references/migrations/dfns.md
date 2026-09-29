# Migrate from Dfns to BlindPay

Keep Dfns for MPC wallet custody and move the stablecoin-to-fiat leg to BlindPay: map transfers and exchange withdrawals to quotes, payouts, and registered external wallets.

Source: https://blindpay.com/docs/migrations/dfns

This guide is for teams running Dfns (docs.dfns.co) for wallet custody and MPC signing who also push stablecoins to fiat through Dfns transfers, an exchange integration (Kraken, Binance, or Coinbase Prime), or a separate on/off-ramp provider wired in alongside Dfns. The migration moves only the stablecoin-to-fiat leg to BlindPay: Dfns keeps custody and signing, while BlindPay takes over payee onboarding, quotes, payouts, and the webhooks that report on them.

## How the concepts map

| Dfns concept | BlindPay concept |
| --- | --- |
| Wallet custody and MPC signing | Stays on Dfns. No BlindPay equivalent, and it is out of scope for this migration. |
| Transfer Asset / broadcast transaction | Payout funding transfer, still signed and broadcast from the Dfns wallet |
| Create Exchange / Create Exchange Withdrawal (Kraken, Binance, Coinbase Prime) | Payout quote (`qu_`) followed by an executed payout (`po_`), or a BlindPay offramp wallet as the deposit destination when there is no quote step |
| Third-party fiat on/off-ramp provider wired in alongside Dfns | BlindPay quotes and payouts, confirmed case by case since Dfns itself does not run fiat rails |
| Payee / counterparty | Customer (`re_`), KYC/KYB run through BlindPay's webhook-driven verification flow |
| Dfns wallet ID used as a funding source | BlindPay external wallet (`bw_`), registered via a sign-message challenge and stored next to the existing Dfns wallet ID |
| Dfns webhook subscription for wallet and policy events | Left untouched on Dfns; BlindPay adds its own Svix-signed webhooks for the fiat leg |
| Stored Dfns transfer or exchange-withdrawal record | `qu_` and `po_` IDs stored alongside the record they replace |

Anything you find with no direct BlindPay equivalent, such as a bespoke ramp provider's settlement logic, gets flagged in the inventory report instead of guessed at.

## Migration steps

### Inventory your Dfns money-movement usage

Search the codebase for every call to Transfer Asset or broadcast transaction on a Dfns wallet, every exchange integration call against Kraken, Binance, or Coinbase Prime, and any third-party fiat ramp wired in alongside Dfns. Also list every Dfns webhook subscription and every stored Dfns wallet ID or exchange account ID. Write this up as a report before touching code: what moves to BlindPay, what stays on Dfns, and anything with no direct BlindPay equivalent.

### Re-onboard payees and register funding wallets

Create [customers](../essentials/customers.md) (`re_`) on BlindPay and run KYC/KYB through BlindPay's flow, driven by `customer.*` webhooks. Compliance status does not carry over from Dfns, so sequence re-verification ahead of cutover. In parallel, register each Dfns-controlled wallet that will fund payouts as a BlindPay external wallet via the sign-message challenge, storing the resulting `bw_` ID next to the existing Dfns wallet ID.

### Rebuild the stablecoin-to-fiat leg on quotes and payouts

Replace Dfns transfers and exchange withdrawals with BlindPay's quote-then-execute model: request a [payout quote](../payouts/payout-quotes.md) (`qu_`), then call the [EVM payout](../payouts/payout-evm.md) endpoint before `expires_at`, signing and broadcasting the funding transfer from the Dfns wallet exactly as before. Store the `qu_` and `po_` IDs alongside the Dfns record they replace. Where a flow just needs stablecoins converted to fiat on arrival with no quote step, evaluate an [offramp wallet](../payouts/offramp-wallets.md) as the deposit destination instead.

### Port fiat-leg notifications to BlindPay webhooks

Add [BlindPay webhooks](../essentials/webhooks.md), verified with `svix-id`, `svix-timestamp`, and `svix-signature` against the raw body using your `whsec_` secret, and dedup on `svix-id`. See [webhook verification](../essentials/webhooks-verification.md) and the [event catalog](../essentials/webhooks-events.md). Leave any existing Dfns webhook subscriptions for wallet and policy events untouched, and confirm signature verification passes against a BlindPay test delivery before relying on it.

### Cut over per flow behind a feature flag

Dual-run each flow with new payouts settling through BlindPay while in-flight Dfns-side transfers or exchange withdrawals finish on the old path. Keep both webhook handlers live during the window, and retire the old fiat-leg path only after one full dual-run settlement cycle clears on BlindPay.

## Copy-paste prompt

Paste this into your coding agent to run the migration end to end.

```text [Migration prompt]
You are migrating the stablecoin-to-fiat leg of my application from Dfns (docs.dfns.co) to the BlindPay API. Dfns stays in place for wallet creation, MPC signing, and custody; only the offramp/payout flow moves to BlindPay.

Before writing code, read these sources and follow them over any prior knowledge:
- https://blindpay.com/docs/llms.txt (read the quickstarts, customers, bank accounts, quotes, wallets, and webhooks pages)
- The OpenAPI spec: curl https://api.blindpay.com/doc

Do the migration in this order:
1. Inventory my Dfns money-movement usage from the codebase: every call to Transfer Asset / broadcast transaction on a Dfns wallet, every exchange integration call (Create Exchange, Create Exchange Withdrawal against Kraken, Binance, or Coinbase Prime), any third-party fiat on/off-ramp provider wired in alongside Dfns (confirm the specific provider in the Dfns docs, since Dfns itself does not run fiat rails), every Dfns webhook subscription, and every stored Dfns wallet ID or exchange account ID. Produce a written inventory report before touching code: what moves to BlindPay, what stays on Dfns, and anything with no direct BlindPay equivalent flagged instead of guessed.
2. Re-onboard payees on BlindPay: create customers (re_) and run KYC/KYB via BlindPay's flow, driven by customer.* webhooks. Compliance status does not carry over from Dfns or any prior ramp provider, so sequence re-verification ahead of cutover. In parallel, register each Dfns-controlled wallet that will fund payouts as a BlindPay external wallet via the sign-message challenge, storing the resulting bw_ ID next to the existing Dfns wallet ID.
3. Rebuild the stablecoin-to-fiat leg on BlindPay's quote-then-execute model: request a quote (qu_), then call POST /v1/instances/{instance_id}/payouts/evm before expires_at, signing and broadcasting the funding transfer from the Dfns wallet exactly as before. Store qu_ and po_ IDs alongside the Dfns transfer or exchange-withdrawal record they replace. Where a flow just needs stablecoins converted to fiat on arrival with no quote step, evaluate a BlindPay offramp wallet as the deposit destination instead.
4. Port fiat-leg notifications to BlindPay's Svix-signed webhooks (svix-id, svix-timestamp, svix-signature, verified with whsec_ against the raw body), with svix-id dedup, while leaving any existing Dfns webhook subscriptions for wallet and policy events untouched. Confirm signature verification passes against BlindPay's test webhook delivery before relying on it.
5. Cut over per flow behind a feature flag: dual-run with new payouts settling through BlindPay while in-flight Dfns-side transfers or exchange withdrawals finish on the old path, keep both webhook handlers live during the window, and retire the old fiat-leg path only after one full dual-run settlement cycle clears on BlindPay.

Constraints:
- Dfns wallets keep custody and signing; do not move private key material or MPC shares.
- Develop against a BlindPay development instance (USDB on testnets, KYC auto-approves, $666.00 forces a failed payout and $777.00 forces a refund for testing) before touching mainnet rails.
- Amounts are integer minor units; BlindPay API keys stay server-side.
- Produce the migration report (inventory, mapping table, gaps, re-KYC plan) before changing code.

Deliverables: the migration report, the external-wallet registration script for existing Dfns wallets, the BlindPay quote-and-payout code path behind a flag, webhook handlers, and a cutover checklist.
```

## Before you cut over

- Build and test against a [development instance](../essentials/sandbox-vs-production.md) first: USDB on testnets, KYC auto-approves, and $666.00 / $777.00 test amounts force a failed payout or a refund.
- Amounts are integer minor units, not decimals. Check every place you format or parse a payout amount.
- BlindPay API keys stay server-side. Never call the API from client code.
- Cut over per flow behind a feature flag, not all at once, so a bad flow doesn't take down every corridor.
- Dual-run each flow for one full settlement cycle, with both Dfns and BlindPay webhook handlers live, before retiring the old fiat-leg path.

## Related docs

- [Customers](../essentials/customers.md)
- [Payout quotes](../payouts/payout-quotes.md)
- [Payout EVM](../payouts/payout-evm.md)
- [Offramp wallets](../payouts/offramp-wallets.md)
- [Webhooks](../essentials/webhooks.md)
