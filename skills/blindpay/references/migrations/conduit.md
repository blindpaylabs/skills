# Migrate from Conduit to BlindPay

Move a Conduit cross-border payments integration to BlindPay: counterparties, corridors, and settlement tracking on the quote-and-execute model.

Source: https://blindpay.com/docs/migrations/conduit

This guide is for teams moving off Conduit's cross-border payments API. Conduit counterparties and recipients map to BlindPay customers with bank accounts typed per rail, and Conduit's cross-border payment calls split into BlindPay's explicit quote-then-execute steps: payin quotes and payins for fiat-to-stablecoin, payout quotes and payouts for stablecoin-to-fiat. The migration re-onboards counterparties for KYC/KYB, rebuilds money movement on the quote model, and cuts over corridor by corridor behind a feature flag.

## How the concepts map

| Conduit concept | BlindPay concept |
| --- | --- |
| Counterparty or recipient | Customer (`re_`) with KYC/KYB |
| Recipient bank account | Bank account (`ba_`) typed per rail: pix, spei, ach, wire, rtp, sepa, faster_payments, transfers |
| Cross-border payment, fiat to stablecoin | Payin quote plus payin (`qu_`, `pi_`) |
| Cross-border payment, stablecoin to fiat | Payout quote plus payout (`qu_`, `po_`) |
| Status updates | Svix-verified `customer.*`, `payout.*`, and `payin.*` webhooks |

Anything in your Conduit usage that has no direct BlindPay equivalent should be flagged during the inventory step below rather than approximated.

## Migration steps

### Inventory your Conduit usage

Go through your codebase and list every Conduit endpoint, webhook, stored ID, and the corridors you move money on. Derive the concept mapping from actual usage rather than assumptions, using the table above as a starting point, and write down anything with no direct BlindPay equivalent instead of guessing at one.

### Verify corridor coverage

List every source and destination currency your integration uses and check it against BlindPay's supported rails and limits. See [supported countries](../kb/supported-countries.md), [payment methods](../kb/payment-methods.md), and [cut-off times and limits](../kb/cut-off-times.md). Call out any gaps before you change code.

### Re-onboard counterparties

Compliance does not transfer between providers, so KYC/KYB re-verification has to happen before cutover. Create each counterparty as a [customer](../essentials/customers.md), following [KYC basics](../kb/kyc-basics.md), [KYC](../kb/kyc.md), and [KYB documents](../kb/kyb-documents.md), and drive the re-onboarding flow off `customer.*` [webhook events](../essentials/webhooks-events.md).

### Rebuild money movement

Replace single-call Conduit payments with BlindPay's quote-then-execute model. Create a [payin quote](../payins/payin-quotes.md) and execute the [payin](../payins/payins.md) for fiat-to-stablecoin, or a [payout quote](../payouts/payout-quotes.md) and [payout](../payouts/payouts.md) for stablecoin-to-fiat, executing before the quote's `expires_at`. Persist the resulting `qu_`, `po_`, and `pi_` IDs alongside the legacy Conduit IDs during the transition, and drive all status from [Svix-verified webhooks](../essentials/webhooks-verification.md).

### Cut over corridor by corridor

Roll out behind a feature flag and dual-run each corridor before retiring the Conduit path for it. Only decommission a corridor once its in-flight settlements have cleared on both sides.

## Copy-paste prompt

Paste this into your coding agent to run the migration end to end.

```text [Migration prompt]
You are migrating my application from the Conduit cross-border payments API to the BlindPay API.

Before writing code, read these sources and follow them over any prior knowledge:
- https://blindpay.com/docs/llms.txt (read the quickstarts, customers, bank accounts, quotes, and webhooks pages)
- The OpenAPI spec: curl https://api.blindpay.com/doc

Do the migration in this order:
1. Inventory my Conduit usage from the codebase: every endpoint, webhook, stored ID, and the corridors we move money on. Derive the concept mapping from actual usage, roughly: Conduit counterparties or recipients map to BlindPay customers (re_) with KYC/KYB plus bank accounts (ba_) typed per rail (pix, spei, ach, wire, rtp, sepa, transfers); cross-border payment or conversion calls map to BlindPay's explicit quote then execute steps (payin quotes plus payins for fiat-to-stablecoin, payout quotes plus payouts for stablecoin-to-fiat). Flag anything with no direct BlindPay equivalent instead of guessing.
2. Verify corridor coverage: list every source and destination currency we use and check it against BlindPay's supported rails and limits; call out gaps before any code changes.
3. Re-onboard counterparties on BlindPay: compliance does not transfer between providers, so sequence KYC/KYB re-verification before cutover, driven by customer.* webhooks.
4. Rebuild money movement: quote, execute before expires_at, persist qu_/po_/pi_ IDs alongside legacy Conduit IDs during the transition, and drive all status from Svix-verified payout.* and payin.* webhooks.
5. Cut over corridor by corridor behind a feature flag with a dual-run window; retire the Conduit path only after in-flight settlements clear.

Constraints:
- Develop against a BlindPay development instance (USDB on testnets) first.
- Amounts are integer minor units; API keys stay server-side.
- Produce a written migration report (mapping table, corridor coverage, gaps, re-KYC plan) before changing code.

Deliverables: the migration report, re-onboarding script, BlindPay code paths behind a flag, webhook handlers, and a cutover checklist.
```

## Before you cut over

- Build and test against a BlindPay development instance (USDB on testnets) before touching production corridors.
- Amounts are integer minor units: check every place you format or parse a Conduit amount.
- Keep API keys server-side; never call BlindPay from client code.
- Have the migration report (mapping table, corridor coverage, gaps, re-KYC plan) reviewed before any code changes ship.
- Dual-run at least one corridor and let in-flight settlements clear before retiring the matching Conduit path.

## Related docs

- [Customers](../essentials/customers.md)
- [Bank accounts](../payouts/bank-accounts.md)
- [Payouts](../payouts/payouts.md)
- [Payins](../payins/payins.md)
- [Webhooks](../essentials/webhooks.md)
