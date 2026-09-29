# Migrate from Fireblocks to BlindPay

Move the stablecoin-to-fiat payout leg of a Fireblocks integration to BlindPay: re-onboard payees, rebuild the quote-then-payout flow, and port webhooks, while Fireblocks stays as custodian.

Source: https://blindpay.com/docs/migrations/fireblocks

This guide is for teams using Fireblocks Payments Engine for the fiat payout and offramp leg of a stablecoin app, who want to move that leg to BlindPay while keeping Fireblocks in place for custody and transaction signing (vault accounts, transactions, whitelisted addresses). The migration replaces Fireblocks payout instruction sets with BlindPay's quote-then-payout flow, re-onboards payees as BlindPay customers with their own bank accounts, and swaps Fireblocks-Signature webhook verification for BlindPay's Svix-signed events. Custody and signing are out of scope: this only touches the fiat payout surface.

## How the concepts map

| Fireblocks concept | BlindPay concept |
|---|---|
| Payee fiat account | Customer (`re_`), with KYC/KYB and bank accounts (`ba_`) |
| Payout instruction set (creation) | Quote (`qu_`) |
| Payout instruction set (execution) | Payout (`po_`) |
| Fireblocks-Signature / Fireblocks-Webhook-Signature verification | Svix-signed webhooks (`customer.*`, `payout.*`, `wallet.inbound`), verified with `whsec_` |
| Fireblocks-controlled wallet funding a payout | External wallet registered via sign-message challenge (`bw_`) |

Vault accounts, transactions, and whitelisted addresses have no BlindPay equivalent. They stay on Fireblocks as the custody and signing layer for this migration. Flag anything else your codebase uses that isn't listed here instead of guessing, and confirm any Fireblocks mechanism against the Fireblocks docs before relying on it.

## Migration steps

### Inventory your Fireblocks usage

Search the codebase for every Payments Engine call (payout instruction set creation and execution), connected fiat accounts, travel rule policy config, and webhook subscriptions keyed on the Fireblocks-Signature or Fireblocks-Webhook-Signature headers. List every stored Fireblocks ID (payment accounts, payee accounts, payout instruction sets) so the mapping in this guide reflects what you actually use, not a generic assumption.

### Re-onboard payees as BlindPay customers

For each Fireblocks payee fiat account, create a BlindPay [customer](../essentials/customers.md), run KYC or KYB through [document upload](../essentials/upload.md), and add rail-specific [bank accounts](../payouts/bank-accounts.md) (Pix, SPEI, SEPA, ACH, wire), driven by [webhooks](../essentials/webhooks.md). KYC does not transfer between providers, so sequence re-verification ahead of cutover so approved payees are ready when you switch. Store the resulting `re_` and `ba_` IDs next to the legacy Fireblocks payee account IDs.

### Rebuild the payout leg on the quote-then-execute model

Replace Fireblocks payout instruction set creation and execution with BlindPay's two-step flow: request a [payout quote](../payouts/payout-quotes.md), then execute the [payout](../payouts/payouts.md) before `expires_at` (about 5 minutes). If funds stay in Fireblocks vault accounts, register the Fireblocks-controlled wallet with BlindPay through the [external-wallet](../payouts/offramp-wallets.md) sign-message challenge (`bw_`) so BlindPay can source funds from it. Store `qu_`, `po_`, and `bw_` IDs alongside the legacy Fireblocks IDs during the transition.

### Port webhooks to BlindPay's Svix events

Swap Fireblocks-Signature RSA verification for BlindPay's [Svix-signed webhooks](../essentials/webhooks.md) (`customer.*`, `payout.*`, `wallet.inbound`). Verify `svix-id`, `svix-timestamp`, and `svix-signature` against the raw request body with your `whsec_` secret, and dedupe on `svix-id`. Check the full list of [webhook events](../essentials/webhooks-events.md) to confirm every Fireblocks event type you currently consume has a BlindPay equivalent before cutting a flow over.

### Cut over per payout corridor

Put the new payout path behind a feature flag and dual-run corridor by corridor: new payouts go through BlindPay while Fireblocks payout instruction sets finish handling in-flight items. Keep both webhook handlers live during the window, and only retire the Fireblocks payout path once a full dual-run settlement cycle has completed and matched the corridor's expected outcomes.

## Copy-paste prompt

Paste this into your coding agent to run the migration end to end.

```text [Migration prompt]
You are migrating my application's stablecoin-to-fiat payout leg from Fireblocks (developers.fireblocks.com) to BlindPay. Fireblocks can stay in place for custody and transaction signing (vault accounts, transactions, whitelisted addresses); this migration targets only the fiat payout and offramp surface.

Before writing code, read these sources and follow them over any prior knowledge:
- https://blindpay.com/docs/llms.txt (read the quickstarts, customers, bank accounts, payout quotes, payouts, and webhooks pages)
- The OpenAPI spec: curl https://api.blindpay.com/doc

Do the migration in this order:
1. Inventory my Fireblocks usage from the codebase: every Payments Engine call (payout instruction set creation, payout execution), connected fiat accounts, travel rule policy config, and webhook subscriptions (Fireblocks-Signature or Fireblocks-Webhook-Signature headers), plus every stored Fireblocks ID (payment accounts, payee accounts, payout instruction sets). Derive the concept mapping from what we actually use, roughly: Fireblocks payee fiat accounts map to BlindPay customers (re_) with KYC/KYB and bank accounts (ba_); a Fireblocks payout instruction set maps to a BlindPay quote (qu_) followed by an executed payout (po_); Fireblocks vault accounts, transactions, and whitelisted addresses have no BlindPay equivalent and stay on Fireblocks as the custody and signing layer. Flag anything with no direct equivalent instead of guessing, and confirm any Fireblocks mechanism I have not explicitly listed against the Fireblocks docs before relying on it.
2. Re-onboard payees on BlindPay: create customer, KYC/KYB via document upload, add rail-specific bank accounts (Pix, SPEI, SEPA, ACH, wire), driven by webhooks. KYC does not transfer between providers, so plan for re-verification and sequence it before the cutover so approved payees are ready when we switch. Store the resulting re_ and ba_ IDs next to the legacy Fireblocks payee account IDs.
3. Rebuild the payout leg on BlindPay's explicit two-step model: request a quote, then execute the payout before expires_at (about 5 minutes), replacing the Fireblocks payout instruction set creation-then-execution flow. If funds keep custody in Fireblocks vault accounts, register the Fireblocks-controlled wallet with BlindPay through the external-wallet sign-message challenge (bw_ ID) so BlindPay can source funds from it. Store qu_, po_, and bw_ IDs alongside the legacy Fireblocks IDs during the transition.
4. Port webhooks to BlindPay's Svix-signed events (customer.*, payout.*, wallet.inbound), verifying svix-id, svix-timestamp, and svix-signature against the raw request body with whsec_ and deduping on svix-id, replacing the Fireblocks-Signature RSA verification. Confirm every Fireblocks event type we currently consume has a BlindPay equivalent before cutting a flow over.
5. Cut over per payout corridor behind a feature flag: dual-run with new payouts on BlindPay while Fireblocks payout instruction sets handle in-flight items, keep both webhook handlers live during the window, and only retire the Fireblocks payout path once a full dual-run settlement cycle has completed and been verified against the corridor's expected outcomes.

Constraints:
- Develop against a BlindPay development instance (USDB on testnets, KYC auto-approves) first, and exercise the $666.00 forced-failed and $777.00 forced-refunded sentinel amounts before touching production traffic.
- Amounts are integer minor units; API keys stay server-side.
- Produce a written migration report before changing code: the mapping table, gaps, and the re-KYC plan.

Deliverables: the migration report, re-onboarding script, the BlindPay client and quote/payout code paths behind a flag, the external-wallet registration flow if custody stays on Fireblocks, webhook handlers, and a cutover checklist.
```

## Before you cut over

- Develop against a BlindPay [development instance](../essentials/instances.md) first: USDB on testnets, KYC auto-approves.
- Exercise the $666.00 forced-failed and $777.00 forced-refunded sentinel amounts before touching production traffic.
- Amounts are integer minor units. API keys stay server-side, never in client code.
- Produce a written migration report before changing code: the mapping table, gaps, and the re-KYC plan.
- Dual-run each corridor through at least one full settlement cycle, including the sentinel failure and refund cases, before retiring the Fireblocks payout path.

## Related docs

- [Customers](../essentials/customers.md)
- [Bank accounts](../payouts/bank-accounts.md)
- [Payout quotes](../payouts/payout-quotes.md)
- [Payouts](../payouts/payouts.md)
- [Webhooks](../essentials/webhooks.md)
