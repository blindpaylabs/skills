# Migrate from Anchorage Digital to BlindPay

Keep Anchorage Digital for custody and move the stablecoin-to-fiat leg to BlindPay: map transfers, withdrawals, and settlement events to BlindPay quotes, payouts, and webhooks.

Source: https://blindpay.com/docs/migrations/anchorage

This guide is for teams that use Anchorage Digital (Atlas settlement, API custody, or Porto self-custody wallets) to hold stablecoins and want to add BlindPay as the fiat offramp, without moving custody or signing off Anchorage. Anchorage stays the custodian: BlindPay only converts stablecoins to fiat and pays out to a bank account. The migration replaces whatever Anchorage transfer or withdrawal call ends a payout flow today with a BlindPay quote-then-payout pair, and replaces Anchorage's webhook or polling logic with BlindPay's signed payout webhooks.

## How the concepts map

| Anchorage concept | BlindPay concept |
| --- | --- |
| Payee or recipient record | Customer (`re_`) with KYC or KYB data |
| Payee bank details | Bank account (`ba_`) for the payout rail (Pix, SPEI, SEPA, ACH, or wire) |
| Anchorage vault or Porto wallet holding funds until payout | External wallet (`bw_`), registered with BlindPay via a sign-message challenge |
| Transfer or withdrawal request | Quote (`qu_`), then a payout (`po_`) executed before the quote expires |
| Transfer webhook or status polling | BlindPay webhook, Svix-signed, updating payout status from payout events |
| Atlas settlement, API custody operations, Porto signing | No BlindPay equivalent. These stay on Anchorage as the custody and signing source of truth |

## Migration steps

### Inventory your Anchorage usage

List every Anchorage transfer, withdrawal, Atlas settlement, or webhook call in the codebase, along with every stored Anchorage ID (vault ID, wallet ID, transfer ID, settlement ID). Separate the flows that stay on Anchorage (custody, signing, on-chain settlement) from the ones that end in a fiat offramp, since only the offramp flows move to BlindPay. Write this inventory down before touching code.

### Onboard payees as BlindPay customers

For each payee, create a [customer](../essentials/customers.md) (`re_`) with KYC or KYB data, then add their payout [bank account](../payouts/bank-accounts.md) (`ba_`) for the correct rail. Store the resulting `re_` and `ba_` IDs against the same record that holds the Anchorage vault or wallet ID today, so the two systems stay linked.

### Register external wallets if funds stay on Anchorage

If stablecoins remain in an Anchorage Porto or vault wallet until payout time, register that wallet with BlindPay as an [external wallet](../payouts/offramp-wallets.md) using the sign-message challenge flow and store the resulting `bw_` ID. This lets BlindPay recognize funds sent from that wallet without taking custody of it.

### Rebuild the offramp on quote then payout

Replace the Anchorage transfer or withdrawal call with BlindPay's two-step model: request a [payout quote](../payouts/payout-quotes.md) (`qu_`) for the amount and rail, then execute it as a [payout](../payouts/payout-evm.md) (`po_`) before the quote's `expires_at`, about five minutes out. Confirm a payout reaches a terminal state end to end against a [development instance](../essentials/sandbox-vs-production.md) before moving on.

### Wire up webhooks

Add BlindPay [webhook](../essentials/webhooks.md) handling alongside the existing Anchorage webhook or polling logic: verify the Svix signature against the raw request body with your `whsec_` signing secret, dedupe on the delivery ID, and update payout status from the [payout events](../essentials/webhooks-events.md). Use the development instance's sentinel amounts to confirm the forced-failed and forced-refunded paths both work.

### Cut over per payout flow

Put each payout flow behind a feature flag and dual-run the Anchorage offramp path and the BlindPay path side by side. Compare outcomes on real payouts, and only remove the old offramp code once a full dual-run cycle settles cleanly on BlindPay, with Anchorage still reconciling as the custody source of truth.

## Copy-paste prompt

Paste this into your coding agent to run the migration end to end.

```text [Migration prompt]
You are adding BlindPay as the fiat offramp for my application. Anchorage Digital stays the custodian and signer for stablecoin balances (Atlas settlement, API custody operations, or Porto self-custody wallets). BlindPay only handles converting those stablecoins to fiat and paying out to a bank account.

Before writing code, read these sources and follow them over any prior knowledge:
- https://blindpay.com/docs/llms.txt (read the quickstarts, customers, bank accounts, quotes, payouts, and webhooks pages)
- The OpenAPI spec: curl https://api.blindpay.com/doc
- https://docs.anchorage.com/knowledge-base/api-reference/introduction for how our existing Anchorage integration authenticates and what it calls

Do the migration in this order:
1. Inventory my Anchorage usage from the codebase: every Anchorage transfer, withdrawal, Atlas settlement, or webhook call, plus every stored Anchorage ID (vault ID, wallet ID, transfer ID, settlement ID). Note which Anchorage flows stay as-is (custody, signing, on-chain settlement) and which ones today end in an offramp to fiat, since only the latter move to BlindPay. Confirm exact endpoint names and payloads in the Anchorage docs rather than assuming, and flag anything with no direct BlindPay equivalent. Produce a written inventory report before touching code.
2. Onboard the receiving parties as BlindPay customers: create a customer (re_) per payee with KYC or KYB data, add their payout bank account (ba_) for the correct rail (Pix, SPEI, SEPA, ACH, or wire), and store the resulting re_ and ba_ IDs against the same record that holds the Anchorage vault or wallet ID today.
3. If stablecoins are staying in an Anchorage Porto or vault wallet until payout time, register that wallet with BlindPay as an external wallet using the sign-message challenge flow and store the resulting bw_ ID, so BlindPay can recognize funds sent from it without taking custody itself.
4. Rebuild the offramp leg on BlindPay's two-step model: request a quote (qu_) for the payout amount and rail, then call POST /v1/instances/{instance_id}/payouts/evm to execute it as a payout (po_) before the quote's expires_at (about 5 minutes out). Replace whatever Anchorage transfer or withdrawal call currently ends a payout flow with this quote-then-execute pair, and confirm a payout reaches a terminal state end to end against a BlindPay development instance (USDB on testnets, KYC auto-approves) before moving on.
5. Add BlindPay webhook handling alongside whatever Anchorage webhook or polling logic exists today: verify the Svix signature (svix-id, svix-timestamp, svix-signature) against the raw request body with your whsec_ signing secret, dedupe on svix-id, and update payout status from the payout events. Use the $666.00 and $777.00 sentinel amounts in the development instance to confirm the forced-failed and forced-refunded paths are both handled.
6. Cut over per payout flow behind a feature flag: dual-run Anchorage's existing offramp path and the new BlindPay path side by side, compare outcomes on real payouts, and only remove the old offramp code once a full dual-run cycle settles cleanly on BlindPay and Anchorage still reconciles as the custody source of truth.

Constraints:
- Anchorage keeps custody and signing for anything not explicitly moved to BlindPay; do not build a second custody path.
- API keys for both providers stay server-side only.
- Amounts are integer minor units on both sides; no floating point money math anywhere in the payout path.
- Develop and test the new path against a BlindPay development instance before touching production keys.

Deliverables: the Anchorage usage inventory report, the customer and bank account onboarding script, the external wallet registration flow (if funds stay in Anchorage until payout), the quote-then-payout code path, webhook handlers with signature verification, and a cutover checklist per payout flow.
```

## Before you cut over

- Run the migration against a [development instance](../essentials/sandbox-vs-production.md) first, and review the inventory report before letting an agent touch onboarding or payout code.
- Provision bank accounts, and external wallets if funds stay on Anchorage, for a small set of test payees before onboarding the rest.
- Keep API keys for both providers server-side only, never in client code.
- Use integer minor units for every amount in the payout path: no floating point money math.
- Move one payout flow at a time behind a feature flag, dual-run it against Anchorage, and only cut the next flow over once the previous one settles cleanly on BlindPay.

## Related docs

- [Payout quickstart](../getting-started/quickstart-payout.md)
- [Offramp wallets](../payouts/offramp-wallets.md)
- [Payout quotes](../payouts/payout-quotes.md)
- [Webhooks](../essentials/webhooks.md)
- [Sandbox vs production](../essentials/sandbox-vs-production.md)
