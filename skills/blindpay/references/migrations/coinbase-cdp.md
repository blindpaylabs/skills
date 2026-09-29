# Migrate from Coinbase CDP to BlindPay

Move the stablecoin-to-fiat offramp leg of a Coinbase Developer Platform integration to BlindPay: add Pix, SPEI, SEPA, and wire payouts while CDP wallets keep custody.

Source: https://blindpay.com/docs/migrations/coinbase-cdp

This guide is for teams using Coinbase Developer Platform's Offramp (session tokens, the hosted sell URL, and offramp webhooks) to cash out stablecoins to fiat. It covers moving only that cash-out leg to BlindPay: CDP's session-token-plus-hosted-sell-URL flow becomes a BlindPay quote and executed payout, CDP's ACH/PayPal/Coinbase-balance cashout methods become BlindPay bank accounts, and Pix, SPEI, SEPA, and wire become available where CDP Offramp had no coverage at all. CDP wallets or CDP Server Wallets can stay in place for custody and signing: this migration does not touch them.

## How the concepts map

| Coinbase CDP concept | BlindPay concept |
| --- | --- |
| Offramp session token + hosted sell URL (pay.coinbase.com sell flow) | Quote (`qu_`) followed by an executed payout (`po_`) |
| Cashout methods: ACH, PayPal, Coinbase balance | Bank account (`ba_`), plus Pix, SPEI, SEPA, and wire, which CDP Offramp does not support |
| Coinbase's own KYC on the end user's Coinbase account | Customer (`re_`) with KYC/KYB via document upload |
| `offramp.transaction.*` webhooks | Svix-signed webhooks: `customer.*`, `payout.*`, `wallet.inbound` |
| `partnerUserRef`, `transactionId`, wallet addresses | Stored alongside the new `re_`, `ba_`, `qu_`, `po_` IDs, not replaced |

CDP wallets and CDP Server Wallets have no BlindPay equivalent. If funds should keep custody in a CDP wallet, register that address with BlindPay as an external wallet (`bw_`) through the sign-message challenge, so BlindPay can source funds from it without moving custody. Flag anything else with no direct equivalent instead of guessing.

## Migration steps

### Inventory your CDP Offramp usage

Find every call to CDP's session token endpoint, the hosted sell URL construction (`sessionToken`, `partnerUserRef`, `defaultCashoutMethod`), the sell quote and transaction-status calls, offramp webhook subscriptions, and any stored CDP identifiers. Confirm exact endpoint paths and payload fields against the CDP docs rather than assuming, then derive your concept mapping from what you actually use. See [SDKs](../getting-started/sdks.md) and the [quickstart for stablecoin to fiat](../getting-started/quickstart-payout.md) for the BlindPay side of that mapping.

### Onboard payees on BlindPay

CDP Offramp lets Coinbase's own KYC on the end user's account stand in for onboarding, so this is new work, not a data migration. Create a customer, run KYC or KYB through document upload, and add rail-specific bank accounts for Pix, SPEI, SEPA, ACH, and wire, driven by webhooks. See [customers](../essentials/customers.md), [KYC basics](../kb/kyc-basics.md), [upload](../essentials/upload.md), and [bank accounts](../payouts/bank-accounts.md). Store the resulting `re_` and `ba_` IDs next to any legacy CDP identifiers.

### Rebuild the cash-out leg on quote then payout

Replace CDP's session-token-plus-hosted-sell-URL flow with BlindPay's explicit two-step model: request a [payout quote](../payouts/payout-quotes.md), then execute the payout before `expires_at`. If funds keep custody in a CDP wallet, register that address with BlindPay through the external-wallet flow described in [offramp wallets](../payouts/offramp-wallets.md) so BlindPay can source funds from it without moving custody. Store `qu_`, `po_`, and `bw_` IDs alongside any legacy CDP transaction references.

### Port webhooks

Move off CDP's `offramp.transaction.*` webhook verification onto BlindPay's Svix-signed events (`customer.*`, `payout.*`, `wallet.inbound`), verifying `svix-id`, `svix-timestamp`, and `svix-signature` against the raw request body with your `whsec_` secret, and dedupe on `svix-id`. Confirm every CDP offramp event type you currently consume has a BlindPay equivalent before cutting a corridor over. See [webhooks](../essentials/webhooks.md), [webhook events](../essentials/webhooks-events.md), and [webhook verification](../essentials/webhooks-verification.md).

### Cut over per corridor

Launch Pix, SPEI, SEPA, and wire on BlindPay immediately, since CDP Offramp has no equivalent for them. Dual-run the ACH corridor: send new payouts through BlindPay while CDP Offramp finishes in-flight items, and keep both webhook handlers live during that window. Only retire the CDP Offramp path for a corridor once a full settlement cycle has completed and been verified against expected outcomes.

## Copy-paste prompt

Paste this into your coding agent to run the migration end to end.

```text [Migration prompt]
You are migrating my application's stablecoin-to-fiat offramp leg from Coinbase Developer Platform (CDP) to BlindPay. CDP wallets (or CDP Server Wallets) can stay in place for custody and signing; this migration targets only the cash-out surface: CDP's Offramp session tokens, hosted sell URLs, and offramp webhooks.

Before writing code, read these sources and follow them over any prior knowledge:
- https://blindpay.com/docs/llms.txt (read the quickstarts, customers, bank accounts, quotes, payouts, offramp wallets, and webhooks pages)
- The OpenAPI spec: curl https://api.blindpay.com/doc

Do the migration in this order:
1. Inventory my CDP Offramp usage from the codebase: the session token call (POST to CDP's onramp/v1/token endpoint), the hosted sell URL construction (pay.coinbase.com sell flow with sessionToken, partnerUserRef, defaultCashoutMethod), the sell quote and transaction-status calls, offramp webhook subscriptions, and any stored CDP identifiers (partnerUserRef, transactionId, wallet addresses). Confirm exact endpoint paths and payload fields against the CDP docs rather than assuming. Derive the concept mapping from what we actually use, roughly: CDP's hosted sell flow (session token plus sell URL) maps to a BlindPay quote (qu_) followed by an executed payout (po_); CDP's ACH/PayPal/Coinbase-balance cashout methods map to BlindPay bank accounts (ba_) but BlindPay also covers Pix, SPEI, SEPA, and wire, which CDP Offramp does not support; CDP wallets or CDP Server Wallets have no BlindPay equivalent and can stay as the custody and signing layer. Flag anything with no direct equivalent instead of guessing.
2. Onboard payees on BlindPay directly: create customer, KYC/KYB via document upload, add rail-specific bank accounts (Pix, SPEI, SEPA, ACH, wire), driven by webhooks. CDP Offramp lets Coinbase's own KYC on the end user's Coinbase account stand in for this, so this is new work, not a data migration; sequence it before cutover so payees are ready when we switch. Store the resulting re_ and ba_ IDs next to any legacy CDP identifiers (partnerUserRef, wallet addresses).
3. Rebuild the cash-out leg on BlindPay's explicit two-step model: request a quote, then execute the payout before expires_at (about 5 minutes), replacing CDP's session-token-plus-hosted-sell-URL flow. If funds keep custody in a CDP wallet, register that CDP-controlled address with BlindPay through the external-wallet sign-message challenge (bw_ ID) so BlindPay can source funds from it without moving custody. Store qu_, po_, and bw_ IDs alongside any legacy CDP transaction references.
4. Port webhooks to BlindPay's Svix-signed events (customer.*, payout.*, wallet.inbound), verifying svix-id, svix-timestamp, and svix-signature against the raw request body with whsec_ and deduping on svix-id, replacing CDP's offramp.transaction.* webhook verification. Confirm every CDP offramp event type we currently consume has a BlindPay equivalent before cutting a corridor over.
5. Cut over per payout corridor behind a feature flag: launch Pix, SPEI, SEPA, and wire on BlindPay immediately since CDP Offramp has no equivalent for them, and dual-run the ACH corridor with new payouts on BlindPay while CDP Offramp handles in-flight items. Keep both webhook handlers live during the ACH dual-run window and only retire the CDP Offramp path for that corridor once a full settlement cycle has completed and been verified against expected outcomes.

Constraints:
- Develop against a BlindPay development instance (USDB on testnets, KYC auto-approves) first, and exercise the $666.00 forced-failed and $777.00 forced-refunded sentinel amounts before touching production traffic.
- Amounts are integer minor units; API keys stay server-side.
- Produce a written migration report before changing code: the mapping table, the corridors CDP Offramp never covered, gaps, and the payee-onboarding plan.

Deliverables: the migration report, the payee onboarding script, the BlindPay client and quote/payout code paths behind a flag, the external-wallet registration flow if custody stays on a CDP wallet, webhook handlers, and a per-corridor cutover checklist.
```

## Before you cut over

- Develop against a BlindPay development instance first: USDB on testnets, KYC auto-approves.
- Exercise the $666.00 forced-failed and $777.00 forced-refunded sentinel amounts before touching production traffic.
- Amounts are integer minor units; keep API keys server-side, never in client code.
- Write the migration report first: the mapping table, the corridors CDP Offramp never covered, remaining gaps, and the payee-onboarding plan.
- Ship Pix, SPEI, SEPA, and wire first since there is nothing to cut over there, then dual-run at least one full ACH settlement cycle, including the sentinel failure and refund cases, before retiring CDP Offramp for that corridor.

## Related docs

- [Customers](../essentials/customers.md)
- [Payout quotes](../payouts/payout-quotes.md)
- [Offramp wallets](../payouts/offramp-wallets.md)
- [Webhooks](../essentials/webhooks.md)
- [Bank accounts](../payouts/bank-accounts.md)
