# Migrate from BitGo to BlindPay

Move the stablecoin-to-fiat leg of a BitGo integration to BlindPay: map wallets, transfers, and settlement webhooks to BlindPay customers, quotes, payouts, and webhook events.

Source: https://blindpay.com/docs/migrations/bitgo

This guide is for teams using BitGo for wallet custody and transaction signing who want to move the offramp and settlement leg, the part that turns stablecoins into fiat, onto BlindPay. BitGo keeps custody and signing: only the send-to-fiat path moves. BitGo wallets become BlindPay external wallets, payout recipients become BlindPay customers with their own bank accounts, and BitGo's send-transaction and settlement calls get rebuilt on BlindPay's quote-then-payout model with Svix webhooks replacing BitGo's wallet webhooks.

## How the concepts map

| BitGo concept | BlindPay equivalent |
| --- | --- |
| BitGo wallet | External wallet (`bw_`), registered via the sign-message challenge flow |
| BitGo wallet ID | `bw_` ID stored against the wallet record |
| Send-transaction / transfer call | Quote (`qu_`) followed by payout execution (`po_`) |
| Transfer ID | `po_` ID |
| Go Network settlement call | Payout execution, `POST /v1/instances/{instance_id}/payouts/evm` |
| Offramp / settlement partner integration | BlindPay payout rails (Pix, SPEI, SEPA, ACH, wire) |
| Payout recipient | Customer (`re_`), onboarded with KYC or KYB |
| Recipient payout rail | Bank account (`ba_`) |
| Wallet webhook (`transfer`, `pendingapproval`, `address_confirmation`) | Svix-signed webhook event, verified with `svix-id`, `svix-timestamp`, `svix-signature` |
| HMAC signature header (`x-signature-sha256`) | `whsec_` secret used to verify the Svix signature |

BitGo's role as custodian and signer has no BlindPay equivalent: BlindPay never takes custody of the underlying assets or signs transactions, it only handles the fiat leg. BitGo qualified-custody and counterparty checks also do not carry over. Any recipient moving to a BlindPay bank account needs fresh KYC or KYB through BlindPay.

## Migration steps

### Inventory your BitGo usage

Go through the codebase for every BitGo wallet call, send-transaction or transfer call, Go Network settlement call, offramp or settlement partner integration, and wallet webhook handler (`transfer`, `pendingapproval`, `address_confirmation`). List every stored BitGo wallet ID and transfer ID. Confirm anything you cannot map with certainty against the BitGo docs, then write down a mapping table like the one above before changing any code.

### Register wallets and onboard customers

Register each existing BitGo wallet as a [blockchain wallet](../payins/blockchain-wallets.md) using the sign-message challenge flow, and store the resulting `bw_` ID against the wallet. Funds stay in BitGo custody: the `bw_` ID is only the connection point BlindPay uses to know where stablecoins come from. Then onboard each payout recipient as a [customer](../essentials/customers.md) with KYC or KYB, and register their payout rail as a [bank account](../payouts/bank-accounts.md) for Pix, SPEI, SEPA, ACH, or wire. Store the `re_` and `ba_` IDs against the corresponding BitGo record. Treat this as new KYC: BitGo's qualified-custody or counterparty checks do not transfer.

### Rebuild settlement on the quote-then-payout model

Replace the BitGo send-transaction and Go Network settlement calls with BlindPay's two-step flow: request a [payout quote](../payouts/payout-quotes.md), then execute the [payout](../payouts/payouts.md) before the quote's `expires_at`, roughly five minutes out. Run a full cycle in a BlindPay development instance first: USDB on testnets, KYC auto-approves, and the `$666.00` forced-failed and `$777.00` forced-refunded sentinels should each produce the expected payout state. See [sandbox vs production](../essentials/sandbox-vs-production.md) for what a development instance covers.

### Port webhooks

Move settlement notifications from BitGo's HMAC-signed wallet webhooks (`x-signature-sha256`) to [BlindPay's webhook events](../essentials/webhooks.md). Verify `svix-id`, `svix-timestamp`, and `svix-signature` against the raw request body using the instance's `whsec_` secret, and deduplicate on `svix-id`. See the [verification guide](../essentials/webhooks-verification.md) and the [event reference](../essentials/webhooks-events.md) for the payout status transitions (`sent`, `failed`, `refunded`) to handle, and make sure each one updates the record BitGo previously used to track that transfer.

### Cut over per corridor

Put the payout flow behind a feature flag and dual-run BitGo and BlindPay settlement side by side for a full cycle, comparing outcomes per transfer. Retire the BitGo settlement path only once every in-flight BitGo transfer has settled and every payout in the dual-run window matches. BitGo keeps holding and signing the underlying assets after cutover: only the fiat leg has moved.

## Copy-paste prompt

Paste this into your coding agent to run the migration end to end.

```text [Migration prompt]
You are migrating the stablecoin-to-fiat payout leg of my application from BitGo to BlindPay. BitGo stays in place for custody and transaction signing: only the offramp and settlement leg moves.

Before writing code, read these sources and follow them over any prior knowledge:
- https://blindpay.com/docs/llms.txt (read the quickstarts, customers, bank accounts, external wallets, quotes, payouts, and webhooks pages)
- The OpenAPI spec: curl https://api.blindpay.com/doc

Do the migration in this order:
1. Inventory my BitGo usage from the codebase: every BitGo wallet call, send-transaction or transfer call, Go Network settlement call, offramp or settlement partner integration, and wallet webhook (transfer, pendingapproval, address_confirmation), plus every stored BitGo wallet ID and transfer ID. Confirm in the BitGo docs anything in the codebase you cannot map with certainty. Produce a written mapping table before changing code: BitGo concept, BlindPay equivalent, and gaps with no direct match.
2. Register my existing BitGo wallets as BlindPay external wallets using the sign-message challenge flow, storing the resulting bw_ ID against each wallet. Funds keep living in BitGo custody; the bw_ ID is the connection point BlindPay uses to know where stablecoins are coming from.
3. Onboard payout recipients as BlindPay customers (re_) with KYC or KYB, and register their payout rails as BlindPay bank accounts (ba_) for Pix, SPEI, SEPA, ACH, or wire as applicable. Store the re_ and ba_ IDs against the corresponding BitGo wallet or account record. Treat this as new KYC: BitGo qualified-custody or counterparty checks do not transfer to BlindPay.
4. Rebuild the settlement leg on BlindPay's two-step model: request a quote (qu_), then execute the payout (po_) via POST /v1/instances/{instance_id}/payouts/evm before the quote's expires_at, roughly five minutes out. Verify a full cycle end to end in a BlindPay development instance: USDB on testnets, KYC auto-approves, and confirm the $666.00 forced-failed and $777.00 forced-refunded sentinels both produce the expected payout state.
5. Port settlement notifications from BitGo's HMAC-signed wallet webhooks (x-signature-sha256) to BlindPay's Svix-signed webhook events, verifying svix-id, svix-timestamp, and svix-signature against the raw request body with the instance's whsec_ secret, and deduplicating on svix-id. Confirm each BlindPay payout status transition (sent, failed, refunded) reaches your handler and updates the record BitGo used to track that transfer.
6. Cut the payout flow over behind a feature flag: dual-run BitGo and BlindPay settlement side by side for a full cycle, compare outcomes per transfer, then retire the BitGo settlement path only once every in-flight BitGo transfer has settled and every payout in the dual-run window matches. BitGo continues to hold and sign the underlying assets after cutover; only the fiat leg has moved.

Constraints:
- BitGo remains the custodian and signer for any wallet not explicitly re-platformed; do not move custody.
- API keys stay server-side, never in client code or logs.
- All amounts are integer minor units; never use floating point for money math.
- Develop and test the full flow against a BlindPay development instance before touching production.

Deliverables: the migration report (mapping table, gaps, re-KYC plan), the external wallet registration script (bw_ IDs), the BlindPay client and quote-then-payout code path behind a flag, Svix webhook handlers with signature verification, and a cutover checklist covering the dual-run comparison and BitGo retirement criteria.
```

## Before you cut over

- Run the full flow, wallet registration, KYC, quote, payout, webhook, against a BlindPay development instance before pointing anything at production.
- Confirm every amount in your code is an integer minor unit: never floating point for money math.
- Keep API keys server-side, never in client code or logs.
- Cut over per corridor behind a feature flag, dual-running BitGo and BlindPay settlement side by side and comparing outcomes per transfer.
- Retire the BitGo settlement path only once every in-flight BitGo transfer has settled and every payout in the dual-run window matches.

## Related docs

- [Blockchain wallets](../payins/blockchain-wallets.md)
- [Payout quotes](../payouts/payout-quotes.md)
- [Payouts](../payouts/payouts.md)
- [Webhooks](../essentials/webhooks.md)
- [Webhooks verification](../essentials/webhooks-verification.md)
