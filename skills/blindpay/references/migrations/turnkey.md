# Migrate from Turnkey to BlindPay

Keep Turnkey for wallets and signing, and move the stablecoin-to-fiat offramp leg of a Turnkey-based product to BlindPay: map payout accounts, quotes, and webhooks.

Source: https://blindpay.com/docs/migrations/turnkey

This guide is for teams that built a stablecoin-to-fiat offramp on top of Turnkey's wallet and signing infrastructure and want to replace the custom payout leg with BlindPay, while Turnkey keeps custody and signing unchanged. It covers onboarding payees as BlindPay customers, registering the Turnkey-controlled wallet as a BlindPay external wallet, and rebuilding payouts on BlindPay's quote-then-execute model. Turnkey's sign transaction and sign raw payload activities, wallet accounts, and policy engine stay in place: only the fiat rail on top of a signed transaction moves.

## How the concepts map

| Turnkey concept | BlindPay concept |
| --- | --- |
| Sign transaction / sign raw payload activity backing a payout | Turnkey wallet stays the signer; BlindPay only consumes funds from the wallet you register |
| Wallet account / address used to fund payouts | External wallet, registered with a sign-message challenge signed by the Turnkey wallet, `bw_` |
| Payee or counterparty record in your custom offramp | Customer, `re_` |
| Payee bank details (Pix, SPEI, SEPA, ACH, wire) | Bank account, `ba_` |
| Custom offramp payout call | Quote (`qu_`) then payout (`po_`), a two-step request-then-execute flow |
| Your own activity webhooks tied to a payout | BlindPay Svix-signed webhooks: `customer.*`, `payout.*`, `wallet.inbound` |

Turnkey's policy engine (consensus and condition rules gating who can sign) has no BlindPay equivalent: it stays entirely on the Turnkey side, since Turnkey keeps signing. Turnkey's own activity webhooks also stay as they are; you only add BlindPay's webhooks alongside them.

## Migration steps

### Inventory your Turnkey usage

List every sign transaction and sign raw payload activity tied to a payout flow, the wallet accounts and addresses those activities sign from, the policies gating them, any activity webhooks you consume, and whatever offramp or fiat rail sits on top of a signed transaction today. Confirm anything you have not explicitly listed against the Turnkey docs before relying on it, and flag anything with no direct BlindPay equivalent instead of guessing.

### Onboard payees as BlindPay customers

Create a customer, run KYC or KYB through document upload, and add rail-specific bank accounts (Pix, SPEI, SEPA, ACH, wire), driven by webhooks. See [customers](../essentials/customers.md), [KYC basics](../kb/kyc-basics.md), and [bank accounts](../payouts/bank-accounts.md). Store the resulting `re_` and `ba_` IDs and produce a written onboarding report before touching money movement.

### Register the Turnkey wallet as an external wallet

Connect the Turnkey wallet to BlindPay without moving funds: register its address through BlindPay's [external wallet](../payouts/offramp-wallets.md) flow using a sign-message challenge signed by the Turnkey wallet, which returns a `bw_` ID. Store that `bw_` ID alongside the Turnkey wallet account it maps to.

### Rebuild payouts on the quote-then-execute model

Replace your custom payout call with BlindPay's explicit two-step flow: request a [payout quote](../payouts/payout-quotes.md) (`qu_`), then execute the [payout](../payouts/payout-evm.md) (`po_`) before `expires_at`, sourcing funds from the Turnkey-controlled wallet registered in the previous step. Store `qu_` and `po_` IDs next to the legacy payout records they replace.

### Port webhooks

Add handlers for BlindPay's [Svix-signed webhooks](../essentials/webhooks.md) covering `customer.*`, `payout.*`, and `wallet.inbound`, verifying `svix-id`, `svix-timestamp`, and `svix-signature` against the raw request body with your `whsec_` secret, and deduping on `svix-id`. See [webhooks verification](../essentials/webhooks-verification.md) and the [webhooks events reference](../essentials/webhooks-events.md). Keep Turnkey's own activity webhooks unchanged, since Turnkey keeps signing.

### Cut over per payout corridor

Cut over behind a feature flag, one payout corridor at a time. Dual-run new payouts on BlindPay while the old offramp path finishes in-flight items, keep both webhook handlers live during the window, and retire the old offramp path only once a full dual-run settlement cycle has completed and matched the corridor's expected outcomes.

## Copy-paste prompt

Paste this into your coding agent to run the migration end to end.

```text [Migration prompt]
You are migrating my application's stablecoin-to-fiat offramp and payout leg from whatever I bolted onto Turnkey (docs.turnkey.com) to BlindPay. Turnkey stays in place as the wallet infrastructure, policy engine, and signing API; my product's stablecoins keep living in Turnkey wallets. This migration targets only the fiat payout surface, not custody or signing.

Before writing code, read these sources and follow them over any prior knowledge:
- https://blindpay.com/docs/llms.txt (read the quickstarts, customers, bank accounts, payout quotes, payouts, external wallets, and webhooks pages)
- The OpenAPI spec: curl https://api.blindpay.com/doc

Do the migration in this order:
1. Inventory my Turnkey usage from the codebase: every sign transaction and sign raw payload activity tied to a payout flow, the wallet accounts and addresses those activities sign from, the policies gating them (consensus and condition rules in the policy engine), any activity webhooks I consume, and whatever offramp or fiat rail I built on top of a signed transaction today (a third-party payout API, a manual process, a liquidation address). Confirm any Turnkey mechanism I have not explicitly listed against the Turnkey docs before relying on it. Flag anything with no direct BlindPay equivalent instead of guessing.
2. Onboard payees on BlindPay: create customer, KYC/KYB via document upload, add rail-specific bank accounts (Pix, SPEI, SEPA, ACH, wire), driven by webhooks. Store the resulting re_ and ba_ IDs, and produce a written onboarding report before touching money movement.
3. Connect the Turnkey wallet to BlindPay without moving funds: register the Turnkey wallet address with BlindPay's external-wallet flow (sign-message challenge signed by the Turnkey wallet, resulting bw_ ID). Store the bw_ ID alongside the Turnkey wallet account it maps to.
4. Rebuild the payout leg on BlindPay's explicit two-step model: request a quote (qu_), then execute the payout (po_) via POST /v1/instances/{instance_id}/payouts/evm before expires_at (about 5 minutes), sourcing funds from the Turnkey-controlled wallet you registered in step 3. Store qu_ and po_ IDs next to the legacy payout records they replace.
5. Port webhooks to BlindPay's Svix-signed events (customer.*, payout.*, wallet.inbound), verifying svix-id, svix-timestamp, and svix-signature against the raw request body with whsec_, and deduping on svix-id. Keep Turnkey's own activity webhooks, if I use them, unchanged since Turnkey keeps signing.
6. Cut over per payout corridor behind a feature flag: dual-run with new payouts on BlindPay while the old offramp path handles in-flight items, keep both webhook handlers live during the window, and retire the old offramp path only once a full dual-run settlement cycle has completed and been verified against the corridor's expected outcomes.

Constraints:
- Develop against a BlindPay development instance (USDB on testnets, KYC auto-approves) first, and exercise the $666.00 forced-failed and $777.00 forced-refunded sentinel amounts before touching production traffic.
- Amounts are integer minor units; API keys stay server-side.
- Produce a written migration report before changing code: the mapping table, gaps, and the onboarding plan.

Deliverables: the migration report, the onboarding script, the Turnkey external-wallet registration flow, the BlindPay client and quote/payout code paths behind a flag, webhook handlers, and a cutover checklist.
```

## Before you cut over

- Develop against a BlindPay development instance first: USDB on testnets, KYC auto-approves.
- Exercise the $666.00 forced-failed and $777.00 forced-refunded sentinel amounts before touching production traffic.
- Amounts are integer minor units, not decimals.
- API keys stay server-side, never in client code.
- Cut over per payout corridor, dual-running both webhook handlers until a full settlement cycle is verified, before retiring the old offramp path for that corridor.

## Related docs

- [Customers](../essentials/customers.md)
- [Bank accounts](../payouts/bank-accounts.md)
- [Payout quotes](../payouts/payout-quotes.md)
- [External wallets](../payouts/offramp-wallets.md)
- [Webhooks verification](../essentials/webhooks-verification.md)
