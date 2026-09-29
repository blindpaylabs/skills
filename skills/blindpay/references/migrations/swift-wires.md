# Migrate from SWIFT wires to BlindPay

Replace multi-day international wires with same-day stablecoin settlement over local rails, keeping SWIFT as a fallback for corridors BlindPay doesn't cover.

Source: https://blindpay.com/docs/migrations/swift-wires

This guide is for teams sending international payouts over SWIFT wires and wanting same-day settlement instead of multi-day MT103 confirmations. It walks through replacing wire initiation with BlindPay stablecoin payouts: beneficiaries become customers with bank accounts on the local rail, wire status tracking becomes webhooks, and the migration happens corridor by corridor so unsupported corridors can stay on SWIFT.

## How the concepts map

| SWIFT wires concept | BlindPay concept |
| --- | --- |
| Beneficiary record | Customer with KYC/KYB |
| IBAN, BIC, and other wire fields | Bank account with rail-specific fields per local rail (Pix for BRL, SPEI for MXN, SEPA for EUR, ACH/RTP/wire for USD, Transfers 3.0 for ARS) |
| Rate lock before sending a wire | Quote |
| Wire initiation | Payout |
| Waiting on an MT103 confirmation | `payout.new`, `payout.update`, and `payout.complete` webhooks |
| Manual reconciliation against SWIFT confirmations | Reconciliation against payout statuses from webhooks |

Corridors BlindPay doesn't cover have no local-rail equivalent: BlindPay also supports `international_swift` as a payment rail, so those beneficiaries can keep moving over SWIFT while the rest of the corridor list migrates to local rails.

## Migration steps

### Inventory the current wire flow

Map where beneficiary bank details are stored today, which currencies and countries you pay, how wire status is tracked, and what reconciliation looks like. Turn this into a corridor-by-corridor mapping to BlindPay rails, flagging any corridor BlindPay doesn't cover so it stays on SWIFT. This mapping drives every step below.

### Migrate beneficiaries to customers and bank accounts

Each beneficiary becomes a [customer](../essentials/customers.md) with KYC/KYB, plus one or more [bank accounts](../payouts/bank-accounts.md) on the local rail for its corridor. Map SWIFT fields like IBAN and BIC to the rail-specific fields documented per bank account type rather than trying to reuse them as-is.

### Build the quote-then-execute payout pipeline

Replace wire initiation with a [quote](../payouts/payout-quotes.md) followed by an executed [payout](../payouts/payouts.md), executed before the quote's `expires_at`. This is the same shape as a wire's rate lock followed by the wire itself, but it settles same-day over local rails instead of over SWIFT.

### Replace status tracking with webhooks

Swap waiting days for MT103 confirmations for `payout.new`, `payout.update`, and `payout.complete` [webhooks](../essentials/webhooks.md), verified with Svix. Drive your own state and reconciliation off these events instead of polling or waiting on SWIFT messages.

### Roll out corridor by corridor

Dual-run one corridor first in shadow mode, comparing costs and settlement times against the existing wire flow. Cut over corridor by corridor behind a feature flag, keeping the SWIFT path as fallback until each corridor is proven, and don't delete the legacy wire code until the rollback window closes.

## Copy-paste prompt

Paste this into your coding agent to run the migration end to end.

```text [Migration prompt]
You are migrating my application's international payments from SWIFT wire transfers to BlindPay stablecoin payouts, which convert USDC/USDT to local fiat and settle over local rails (Pix for BRL, SPEI for MXN, SEPA for EUR, ACH/RTP/wire for USD, Transfers 3.0 for ARS).

Before writing code, read these sources and follow them over any prior knowledge:
- https://blindpay.com/docs/llms.txt (read the payout quickstart, bank accounts, quotes, and payment methods pages)
- The OpenAPI spec: curl https://api.blindpay.com/doc

Do the migration in this order:
1. Inventory my current wire flow: where beneficiary bank details are stored, which currencies and countries we pay, how status is tracked, and what reconciliation looks like today. Produce a corridor-by-corridor mapping to BlindPay rails, flagging any corridor BlindPay does not cover so it stays on SWIFT (BlindPay also supports international_swift as a payment rail where needed).
2. Map beneficiary records to BlindPay: each payee becomes a customer (with KYC/KYB) plus one or more bank accounts with the local rail type. SWIFT fields like IBAN/BIC map to the rail-specific fields documented per bank account type.
3. Replace the wire initiation code path with quote (POST /v1/instances/{instance_id}/quotes) then payout (POST /v1/instances/{instance_id}/payouts/evm), executing before the quote's expires_at.
4. Replace status tracking: instead of waiting days for MT103 confirmations, drive state from payout.new, payout.update, and payout.complete webhooks with Svix signature verification.
5. Plan a phased rollout: dual-run one corridor first (shadow mode comparing costs and settlement times), then cut over corridor by corridor behind a feature flag, keeping the SWIFT path as fallback until each corridor is proven.

Constraints:
- Amounts in integer minor units; API keys server-side only.
- Build and test everything against a development instance (USDB on testnets) before touching production.
- Do not delete the legacy wire code until the rollback window closes.

Deliverables: the corridor mapping document, beneficiary migration script, the new payout code path behind a feature flag, webhook handlers, and a cutover checklist.
```

## Before you cut over

- Build and test everything against a development instance (USDB on testnets) before touching production.
- Keep amounts in integer minor units and API keys server-side only.
- Dual-run one corridor first in shadow mode, comparing costs and settlement times, before cutting over the rest.
- Cut over corridor by corridor behind a feature flag, keeping SWIFT as fallback for corridors not yet proven or not covered at all.
- Don't delete the legacy wire code until the rollback window closes.

## Related docs

- [Payout quickstart](../getting-started/quickstart-payout.md)
- [Bank accounts](../payouts/bank-accounts.md)
- [Payout quotes](../payouts/payout-quotes.md)
- [Payment methods](../kb/payment-methods.md)
- [SWIFT deliverability](../kb/swift-deliverability.md)
