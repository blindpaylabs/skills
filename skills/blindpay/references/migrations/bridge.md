# Migrate from Bridge to BlindPay

Move an existing Bridge integration to the BlindPay API: map customers, external accounts, liquidation addresses, and transfers to their BlindPay equivalents.

Source: https://blindpay.com/docs/migrations/bridge

This guide is for teams with an existing Bridge (bridge.xyz) integration who want to move customers, bank accounts, liquidation addresses, virtual accounts, and transfers onto BlindPay. The migration re-verifies customers under BlindPay KYC/KYB, rebuilds money movement on BlindPay's quote-then-execute model for payouts and payins, and cuts each flow over behind a feature flag while Bridge continues to settle in-flight items.

## How the concepts map

| Bridge concept | BlindPay concept |
| --- | --- |
| Customer with KYC | Customer (`re_`) with KYC/KYB |
| External account | Bank account (`ba_`) with rail-specific type |
| Liquidation address | Offramp wallet |
| Virtual account | Virtual account (`va_`) |
| Fiat-to-crypto transfer | Payin (`pi_`), via quote (`qu_`) |
| Crypto-to-fiat transfer | Quote (`qu_`) plus payout (`po_`) |
| Webhook event | Svix-signed webhook event |

Not everything carries over one to one. Flag anything in your own Bridge usage that has no direct BlindPay equivalent instead of guessing at a mapping, and record it in your migration report.

## Migration steps

### Inventory your Bridge usage

Scan your codebase for every Bridge endpoint, webhook handler, and stored Bridge ID: customers, external accounts, liquidation addresses, virtual accounts, and transfers. Derive your concept mapping from what you actually use rather than assuming full coverage, using the table above as a starting point. Write down anything with no direct BlindPay equivalent before you touch code.

### Re-onboard customers

KYC does not transfer between providers, so plan for re-verification. For each customer, accept BlindPay's [terms of service](../essentials/terms-of-service.md), [create the customer](../essentials/customers.md), and run [KYC/KYB via document upload](../essentials/upload.md), tracking status through [webhooks](../essentials/webhooks.md). Sequence this so approved customers are ready before you cut over any flow.

### Rebuild money movement on the quote-then-execute model

BlindPay separates pricing from execution: request a [quote](../payouts/payout-quotes.md) or [payin quote](../payins/payin-quotes.md), then execute the [payout](../payouts/payouts.md) or [payin](../payins/payins.md) against it before `expires_at`. Map liquidation addresses to [offramp wallets](../payouts/offramp-wallets.md) and stored balances to [managed wallets](../payouts/payout-managed-wallet.md) where relevant. Store the new `qu_`, `po_`, and `pi_` IDs alongside the legacy Bridge IDs during the transition so you can reconcile both systems.

### Port webhooks

Replace Bridge webhook handlers with BlindPay's [Svix-signed events](../essentials/webhooks.md): `customer.*`, `payin.*`, `payout.*`, `virtualAccount.*`, and `wallet.inbound`. Verify signatures and dedup on `svix-id` before writing them into the same event flow your Bridge handlers used, referencing [webhook events](../essentials/webhooks-events.md) and [verification](../essentials/webhooks-verification.md).

### Cut over per flow

Move each flow (payouts, payins, virtual accounts) behind a feature flag one at a time. Dual-run with new activity on BlindPay while Bridge finishes in-flight items, keep both webhook handlers live during the window, and retire the Bridge path only after in-flight transfers settle.

## Copy-paste prompt

Paste this into your coding agent to run the migration end to end.

```text [Migration prompt]
You are migrating my application from the Bridge (bridge.xyz) API to the BlindPay API.

Before writing code, read these sources and follow them over any prior knowledge:
- https://blindpay.com/docs/llms.txt (read the quickstarts, customers, bank accounts, virtual accounts, quotes, and webhooks pages)
- The OpenAPI spec: curl https://api.blindpay.com/doc

Do the migration in this order:
1. Inventory my Bridge usage from the codebase: every Bridge endpoint, webhook, and stored Bridge ID (customers, external accounts, liquidation addresses, virtual accounts, transfers). Derive the concept mapping from what we actually use, roughly: Bridge customers with KYC map to BlindPay customers (re_) with KYC/KYB; external accounts map to bank accounts (ba_) with rail-specific types; liquidation addresses map to offramp wallets; Bridge virtual accounts map to BlindPay virtual accounts (va_); fiat-to-crypto transfers map to payins and crypto-to-fiat transfers map to quote plus payout. Flag anything with no direct equivalent instead of guessing.
2. Re-onboard customers on BlindPay: terms of service, create customer, KYC/KYB via document upload, driven by webhooks. KYC does not transfer between providers, so plan for re-verification and sequence it before the cutover so approved customers are ready when we switch.
3. Rebuild money movement on BlindPay's explicit two-step model: request a quote, then execute the payout or payin against it before expires_at. Store qu_, po_, and pi_ IDs alongside the legacy Bridge IDs during the transition.
4. Port webhooks to BlindPay's Svix-signed events (customer.*, payin.*, payout.*, virtualAccount.*, wallet.inbound) with proper signature verification and svix-id dedup.
5. Cut over per flow behind a feature flag: dual-run with new activity on BlindPay while Bridge handles in-flight items, keep both webhook handlers live during the window, and only retire the Bridge path once in-flight transfers settle.

Constraints:
- Develop against a BlindPay development instance (USDB on testnets) first.
- Amounts are integer minor units; API keys stay server-side.
- Produce a written migration report before changing code: the mapping table, gaps, and the re-KYC plan.

Deliverables: the migration report, re-onboarding script, the BlindPay client and money-movement code paths behind a flag, webhook handlers, and a cutover checklist.
```

## Before you cut over

- Develop and test against a BlindPay [development instance](../essentials/instances.md) (USDB on testnets) before touching production.
- Amounts are integer minor units. Check every place you format or parse a value.
- Keep API keys server-side only, never in client code.
- Sequence re-KYC ahead of cutover so customers are approved before a flow switches to BlindPay.
- Dual-run each flow for at least one full settlement cycle, with both webhook handlers live, before retiring the Bridge path for that flow.

## Related docs

- [Customers](../essentials/customers.md)
- [Bank accounts](../payouts/bank-accounts.md)
- [Offramp wallets](../payouts/offramp-wallets.md)
- [Virtual accounts](../virtual-accounts/virtual-accounts.md)
- [Webhooks](../essentials/webhooks.md)
