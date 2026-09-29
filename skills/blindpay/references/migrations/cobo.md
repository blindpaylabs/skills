# Migrate from Cobo to BlindPay

Move the stablecoin-to-fiat leg of a Cobo Payments integration to BlindPay: keep Cobo for wallet custody if you want, and replace top-up addresses, order mode, and payout destinations with customers, quotes, and payouts.

Source: https://blindpay.com/docs/migrations/cobo

This guide is for teams using Cobo Payments (top-up addresses, payment orders, batch payouts) who want to move the fiat leg to BlindPay while optionally keeping wallets in Cobo custody. It maps Cobo's order-based flow to BlindPay's quote-then-execute model, walks through re-onboarding counterparties under fresh KYC, and covers cutting over per corridor with a dual-run window. Cobo can keep holding and moving the stablecoin; BlindPay takes over converting it to fiat and back.

## How the concepts map

| Cobo concept | BlindPay concept |
|---|---|
| Payment order / order mode | Payin (`pi_`) or payout (`po_`), created from a quote (`qu_`) |
| Top-up address | Virtual account, or an offramp wallet (auto-converting deposit address) |
| Payout destination | Bank account (`ba_`), scoped to a customer (`re_`) and a rail (Pix, SPEI, SEPA, ACH, or wire) |
| Batch payouts | Individual payout calls per destination |
| Refund links | No direct equivalent; flag in your migration report rather than assuming a mapping |
| Subscriptions | No direct equivalent; flag in your migration report rather than assuming a mapping |
| Wallet ID (MPC or custodial) | External wallet (`bw_`), registered via a sign-message challenge, if Cobo keeps custody |
| Signed webhook callbacks | Svix-signed webhook events, verified with the instance's `whsec_` secret |

Refund links and subscriptions have no BlindPay equivalent: flag them in your migration report rather than assuming a mapping, and rebuild that logic in your own application.

### Inventory your Cobo usage

List every Cobo Payments endpoint you call (top-up addresses, payment orders, subscriptions, refund links, payout destinations, batch payouts), every Cobo webhook event you handle, and every stored Cobo ID. Confirm exact endpoint names and event payloads against Cobo's own docs rather than assuming, and write down anything with no direct BlindPay equivalent before touching code. See [webhooks events](../essentials/webhooks-events.md) for the shape of what you're replacing.

### Re-onboard counterparties

KYC does not transfer between providers, so every payer or payee needs a fresh BlindPay pass. Create a [customer](../essentials/customers.md) (`re_`) with KYC or KYB for each counterparty, then register their payout destination as a [bank account](../payouts/bank-accounts.md) (`ba_`) on the correct rail. Store the new `re_` and `ba_` IDs alongside the existing Cobo IDs so both systems stay queryable during cutover.

### Register wallets, if Cobo keeps custody

If wallets stay with Cobo, register each Cobo wallet address as a BlindPay [blockchain wallet](../payins/blockchain-wallets.md) using the sign-message challenge flow, and store the resulting `bw_` ID. Cobo continues to hold and move the stablecoin; BlindPay only handles the conversion to and from fiat.

### Rebuild money movement on quotes

Replace Cobo's order and payout flow with BlindPay's explicit two-step model: request a [payout quote](../payouts/payout-quotes.md) (`qu_`), then execute the [payout](../payouts/payouts.md) (`po_`) before the quote expires, around five minutes. Do the equivalent for inbound flows with [payins](../payins/payins.md), [virtual accounts](../virtual-accounts/virtual-accounts.md), or [offramp wallets](../payouts/offramp-wallets.md) in place of Cobo's top-up addresses. Confirm each flow settles end to end against a [development instance](../essentials/instances.md) before moving to the next one.

### Port webhooks

Move from Cobo's signed callbacks to BlindPay's Svix-signed events: verify `svix-id`, `svix-timestamp`, and `svix-signature` against the raw request body with the instance's `whsec_` secret, and dedup on `svix-id`. Confirm a real delivery arrives and verifies for each event type you rely on, payout completed, payin received, customer KYC status, before depending on it in production. See [webhooks](../essentials/webhooks.md) and [webhook verification](../essentials/webhooks-verification.md).

### Cut over per flow

Run Cobo and BlindPay side by side behind a feature flag for one flow at a time, compare outcomes on a sample of live transfers, and only retire the Cobo path once a full dual-run cycle settles cleanly on both sides with matching amounts and statuses.

## Copy-paste prompt

Paste this into your coding agent to run the migration end to end.

```text [Migration prompt]
You are migrating the stablecoin-to-fiat payment leg of my application from Cobo Payments to the BlindPay API. Cobo may keep custody of wallets (MPC or custodial) and I may keep using it for that; this migration replaces the payout/offramp flow, not necessarily the wallet layer.

Before writing code, read these sources and follow them over any prior knowledge:
- https://blindpay.com/docs/llms.txt (read the quickstarts, customers, bank accounts, virtual accounts, quotes, payouts, offramp wallets, and webhooks pages)
- The OpenAPI spec: curl https://api.blindpay.com/doc

Do the migration in this order:
1. Inventory my Cobo usage from the codebase: every Cobo Payments endpoint I call (top-up addresses, payment orders, subscriptions, refund links, payout destinations, batch payouts), every Cobo webhook event I handle (order status changes, top-up arrivals, payout completions), and every stored Cobo ID (order ID, top-up address, payout destination, wallet ID). Confirm exact endpoint names and event payloads in the Cobo docs (manuals.cobo.com and the Cobo Developer Hub) rather than assuming; flag anything with no direct BlindPay equivalent instead of guessing. Produce a written mapping table plus a gaps list before touching code.
2. Re-onboard my counterparties on BlindPay: create a BlindPay customer (re_) with KYC or KYB per payer or payee, and register their payout destination as a BlindPay bank account (ba_) for the correct rail (Pix, SPEI, SEPA, ACH, or wire). KYC does not transfer between providers, so every counterparty needs a fresh BlindPay KYC/KYB pass. Store the resulting re_ and ba_ IDs alongside the existing Cobo IDs so both systems can be queried during cutover.
3. If wallets stay in Cobo custody, register each Cobo wallet address as a BlindPay external wallet using the sign-message challenge flow, and store the resulting bw_ ID. This lets funds keep living in Cobo-controlled wallets while BlindPay handles the fiat leg: Cobo continues to hold and move the stablecoin, and BlindPay converts it to fiat on payout.
4. Rebuild money movement on BlindPay's explicit two-step model in place of Cobo's order/payout flow: request a quote (qu_), then execute the payout (po_) via POST /v1/instances/{instance_id}/payouts/evm before the quote's expires_at (about 5 minutes). Do the same for inbound flows using payins (pi_), virtual accounts (va_), or offramp wallets (auto-converting deposit addresses) in place of Cobo's top-up addresses and order mode. Confirm each transfer settles end to end against a BlindPay development instance before moving to the next flow.
5. Port webhooks from Cobo's signed callbacks to BlindPay's Svix-signed events: verify svix-id, svix-timestamp, and svix-signature against the raw request body using the instance's whsec_ secret, and dedup on svix-id. Confirm a real webhook delivery is received and verified for each event type you rely on (payout completed, payin received, customer KYC status) before relying on it in production.
6. Cut over per flow behind a feature flag: dual-run Cobo and BlindPay for the same flow, compare outcomes on a sample of live transfers, and only retire the Cobo payment path once a full dual-run cycle settles cleanly on both sides with matching amounts and statuses.

Constraints:
- Develop against a BlindPay development instance (USDB on testnets) first; KYC auto-approves there, and sentinel amounts $666.00 (forced failed) and $777.00 (forced refunded) let you test failure handling without waiting on real bank rails.
- Amounts are integer minor units on both sides of every conversion; never do money math in floating point.
- BlindPay API keys stay server-side only, never exposed to a client.
- Produce a written migration report before changing code: the Cobo-to-BlindPay mapping table, the gaps list, and the re-KYC/re-onboarding plan per counterparty.

Deliverables: the migration report, the counterparty re-onboarding script (re_, ba_, bw_ creation), the BlindPay client and money-movement code paths behind a flag, the Svix webhook handlers, and a cutover checklist per flow.
```

## Before you cut over

- Build and test against a [development instance](../essentials/instances.md) first: KYC auto-approves there, and sentinel amounts $666.00 (forced failed) and $777.00 (forced refunded) let you test failure handling without waiting on real bank rails.
- Do all money math in integer minor units on both sides of every conversion, never floating point.
- Keep BlindPay API keys server-side only, never exposed to a client.
- Write the migration report, the Cobo-to-BlindPay mapping table, the gaps list, and the re-KYC/re-onboarding plan, before you change any code.
- Cut over one flow at a time behind a feature flag, dual-running Cobo and BlindPay until a full cycle settles cleanly on both sides with matching amounts and statuses.

## Related docs

- [Payout quickstart](../getting-started/quickstart-payout.md)
- [Offramp wallets](../payouts/offramp-wallets.md)
- [Webhooks](../essentials/webhooks.md)
- [Customers](../essentials/customers.md)
- [Development instances](../essentials/instances.md)
