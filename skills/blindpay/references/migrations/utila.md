# Migrate from Utila to BlindPay

Keep Utila as custodian and signer, and route the stablecoin-to-fiat leg through BlindPay: map vaults, wallets, transactions, and webhooks to their BlindPay equivalents.

Source: https://blindpay.com/docs/migrations/utila

This guide is for teams running custody and transaction signing on Utila (vaults, wallets, policy engine, co-signer) who want to move only the stablecoin-to-fiat leg to BlindPay. Utila keeps custody and signs every transfer. BlindPay takes over quoting, payout execution, recipient KYC, and fiat settlement, connected back to the Utila wallet through a registered external wallet.

## How the concepts map

| Utila concept | BlindPay concept |
| --- | --- |
| Wallet (custody address) | External wallet (`bw_`), registered via sign-message challenge |
| Vault, transaction initiation, policy engine, co-signer | Stays on Utila. No BlindPay equivalent: custody and signing never move. |
| Address book entry tagged for offramp | Recipient bank account (`ba_`) once the recipient is onboarded |
| Recipient KYC / policy approval | BlindPay customer (`re_`) with its own KYC/KYB |
| `transactions_initiatetransaction` (fiat leg) | Quote (`qu_`) then payout (`po_`), BlindPay's two-step model |
| Transaction resource name | Stored alongside `qu_` and `po_` for cross-system traceability |
| Webhook (`x-utila-signature`, `TRANSACTION_STATE_UPDATED`) | Svix-signed webhooks (`customer.*`, `payout.*`, `wallet.inbound`) |

Utila's KYC and policy approvals do not transfer to BlindPay. Recipients need to go through BlindPay's own KYC/KYB before their first payout.

## Migration steps

### Inventory your Utila usage

List every Utila endpoint your codebase calls (vaults, wallets, `transactions_initiatetransaction`, `transactions_voteontransactionrequest`, address book entries) and every stored Utila resource name. Separate transactions that are actually fiat payouts, transfers to OTC desks, exchange addresses, or offramp-tagged address book entries, from internal treasury moves. Confirm your account's policy engine vote threshold and co-signer behavior directly in the Utila docs rather than assuming, since these are account-configurable. Write down what stays on Utila and what moves to BlindPay before touching code.

### Register your Utila wallets as external wallets

For each Utila wallet address that keeps custody, register it with BlindPay as an [external wallet](../payouts/offramp-wallets.md) using the sign-message challenge flow, and store the resulting `bw_` ID next to the Utila wallet resource name. Funds keep living in Utila wallets. BlindPay only needs proof of ownership to route payouts against them, so this registration is the connection point between the two systems.

### Onboard recipients as BlindPay customers

Create a [customer](../essentials/customers.md) (`re_`) for each recipient and run BlindPay [KYC/KYB](../kb/kyc-basics.md), since Utila approvals do not carry over. Add each recipient's rail-specific [bank account](../payouts/bank-accounts.md) (`ba_`) for Pix, SPEI, SEPA, ACH, or wire. Sequence this re-verification ahead of cutover so no payout is blocked waiting on KYC mid-migration.

### Rebuild the payout leg on quotes and payouts

Replace the fiat side of `transactions_initiatetransaction` with BlindPay's two-step model: request a [quote](../payouts/payout-quotes.md) (`qu_`), then execute the [payout](../payouts/payouts.md) (`po_`) against the registered external wallet before `expires_at` expires, about five minutes. The actual on-chain transfer still goes through Utila's transaction initiation and policy engine vote. Store the `qu_` and `po_` IDs alongside the originating Utila transaction resource name so both sides of one payout stay traceable.

### Port your event handling to BlindPay webhooks

Add handlers for BlindPay's [Svix-signed webhooks](../essentials/webhooks.md) (`customer.*`, `payout.*`, `wallet.inbound`) with [signature verification](../essentials/webhooks-verification.md) and `svix-id` dedup. See the [event reference](../essentials/webhooks-events.md) for the full payload list. Run these handlers alongside your existing Utila webhook handler (`x-utila-signature`, `TRANSACTION_STATE_UPDATED`) so custody-side and fiat-side state both stay current during the migration.

### Cut over per flow, dual-run first

Cut over flow by flow behind a feature flag: new payouts go through the BlindPay quote and payout path while Utila continues to sign and enforce policy on every transfer. Keep both webhook handlers live for the whole dual-run window. Retire the old direct-to-OTC or direct-to-exchange transaction path only after a full dual-run settlement cycle completes with no discrepancies.

## Copy-paste prompt

Paste this into your coding agent to run the migration end to end.

```text [Migration prompt]
You are migrating the stablecoin-to-fiat leg of my application from Utila (docs.utila.io) to BlindPay. Utila stays in place as custodian and transaction signer (vaults, wallets, transaction initiation, policy engine, co-signer). Only the payout and offramp flow moves to BlindPay.

Before writing code, read these sources and follow them over any prior knowledge:
- https://blindpay.com/docs/llms.txt (read the quickstarts, customers, bank accounts, quotes, payouts, offramp wallets, and webhooks pages)
- The OpenAPI spec: curl https://api.blindpay.com/doc

Do the migration in this order:
1. Inventory my Utila usage from the codebase: every Utila endpoint called (vaults, wallets, transactions_initiatetransaction, transactions_voteontransactionrequest, addressbook entries) and every stored Utila resource name (vault, wallet, transaction). Separate transactions that are actually fiat payouts (transfers to OTC desks, exchange addresses, or address book entries tagged for offramp) from internal treasury moves. Confirm the exact policy engine vote-threshold and co-signer behavior in the Utila docs rather than assuming, since these are account-configurable. Produce a written report listing what stays on Utila (custody, signing, policy engine) and what moves to BlindPay (the fiat leg), before touching code.
2. Register each Utila wallet address that will keep custody with BlindPay as an external wallet, using the sign-message challenge flow, and store the resulting bw_ ID next to the Utila wallet resource name. This is the connection point: funds keep living in Utila wallets, BlindPay only needs proof of ownership to route payouts against them.
3. Onboard recipients as BlindPay customers (re_) with KYC/KYB and add their rail-specific bank accounts (ba_) for Pix, SPEI, SEPA, ACH, or wire. Utila KYC and policy approvals do not transfer, so sequence re-verification ahead of cutover.
4. Rebuild the payout leg on BlindPay's two-step model: request a quote (qu_), then execute the payout (po_) referencing the registered external wallet before expires_at (about 5 minutes). The actual on-chain transfer still goes through Utila's transaction initiation and policy engine vote. Store qu_ and po_ IDs alongside the originating Utila transaction resource name so both sides of one payout are traceable.
5. Port event handling to BlindPay's Svix-signed webhooks (customer.*, payout.*, wallet.inbound) with signature verification and svix-id dedup, and run these handlers alongside the existing Utila webhook handler (x-utila-signature, TRANSACTION_STATE_UPDATED) so custody-side and fiat-side state both stay current.
6. Cut over per flow behind a feature flag: dual-run with new payouts going through the BlindPay quote and payout path while Utila continues to sign and enforce policy on every transfer, keep both webhook handlers live during the window, and retire the old direct-to-OTC or direct-to-exchange transaction path only once a full dual-run settlement cycle completes with no discrepancies.

Constraints:
- Develop against a BlindPay development instance (USDB on testnets) first; test the $666.00 forced-failed and $777.00 forced-refunded sentinels before touching production amounts.
- Amounts are integer minor units on both sides; no floating point money math.
- API keys stay server-side.
- Produce the written inventory and mapping report before changing code.

Deliverables: the migration report, the external wallet registration script and stored bw_ mapping, the BlindPay client and quote/payout code paths behind a flag, both webhook handlers, and a cutover checklist.
```

## Before you cut over

- Develop against a BlindPay development instance first, using USDB on testnets, and test the $666.00 forced-failed and $777.00 forced-refunded sentinels before touching production amounts.
- Treat amounts as integer minor units on both sides. No floating point money math.
- Keep API keys server-side only.
- Write the inventory and mapping report, and get it reviewed, especially which vaults and wallets stay on Utila and the re-KYC sequencing, before approving any code changes.
- Dual-run at least one full settlement cycle with both webhook handlers live before retiring the old Utila-to-OTC path.

## Related docs

- [Introduction](../getting-started/introduction.md)
- [Customers](../essentials/customers.md)
- [Offramp wallets](../payouts/offramp-wallets.md)
- [Webhooks](../essentials/webhooks.md)
- [Payout quotes](../payouts/payout-quotes.md)
