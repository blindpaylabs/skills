# Cut-off times

ACH, wire, and SWIFT cut-offs and settlement, instant rails, quote expiry windows, onboarding SLAs, and refund timing.

Source: https://blindpay.com/docs/kb/cut-off-times

Requests submitted after a payment method's daily cut-off are processed the next business day. Business days exclude weekends, US federal holidays, and applicable international banking holidays.

**Note:**

The cut-off and settlement windows below describe the underlying ACH, wire, and SWIFT processing networks in general. They are not the same as BlindPay's end-to-end ETAs for a specific payin or payout, which also include BlindPay's own processing time. For payin arrival windows see [Bank transfer in](../payins/payins.md); for payout ETAs by bank account type see [Pay out to bank](../payouts/payouts.md).

## ACH

| Method | Cut-off (ET) | Estimated settlement |
| --- | --- | --- |
| ACH | 9:00 PM | 1-3 business days |
| Same-Day ACH | 3:00 PM | Same business day |

**Note:**

For US payments, customers with an enabled [virtual account](../virtual-accounts/virtual-accounts.md) see their own account details displayed to the payer. Customers without a virtual account get a unique `memo_code` plus BlindPay's bank account details for the transaction. The memo code is ignored once a virtual account is approved.

## Wire, SWIFT, and TED

| Type | Cut-off (ET) | Estimated settlement |
| --- | --- | --- |
| Domestic wire | 3:00 PM | Same business day |
| International SWIFT | 10:30 AM | Up to 5 business days |
| TED (Brazil) | End of Brazilian banking day (local time) | Same banking day, or the next banking day if submitted after the cut-off, which can push settlement past a weekend or holiday |

ACH, wire, RTP, and SWIFT payouts all carry the same extra country-conditional required fields (a NAICS `business_industry` code for business accounts, a local `tax_id` for the beneficiary's country, and `phone_number` for certain countries), not just SWIFT. See [Bank accounts](../payouts/bank-accounts.md#compliance-fields-for-ach-wire-rtp-and-swift) for the full field list per country.

## Boleto payables

Boletos clear nationally on Brazilian banking days, 06:30 to 18:30 BRT. Outside that window, or on a non-banking day, the payout that pays the boleto is deferred to the next banking day instead of executing right away.

| Payable amount | Same-day cutoff |
| --- | --- |
| Up to R$250,000 | 18:30 BRT |
| Above R$250,000 | 14:30 BRT |

A boleto quoted before 06:30 still pays the same day; the rail just hasn't opened yet. One quoted after its cutoff, or on a weekend or Brazilian bank holiday, is scheduled for the next banking day, so `payable.complete` and `payout.complete` land later than they would for PIX, which runs 24/7 and is unaffected by any of this.

**Warning:**

A boleto payout that lands on a compliance hold (see [Compliance holds](#compliance-holds) below) can wait up to 30 days before it releases. If the boleto's due date falls before the day the payment can then actually execute, requoting the same payable fails with `payable_boleto_would_be_overdue` rather than sending out a boleto that can no longer be paid. Don't assume a payable that quoted successfully days ago still can; requote close to execution instead of caching an old quote.

## Instant methods

These rails settle in minutes rather than business days, both for payins (money coming in) and payouts (money going out):

| Method | Currency | Country | Payin arrival | Payout `type` value |
| --- | --- | --- | --- | --- |
| Pix | BRL | Brazil | Up to 5 minutes | `pix` |
| PIX Safe | BRL | Brazil | Payout only | `pix_safe` |
| SPEI (CLABE) | MXN | Mexico | Up to 10 minutes | `spei_bitso` |
| Transfers (CBU) | ARS | Argentina | Up to 10 minutes | `transfers_bitso` |
| PSE | COP | Colombia | Up to 10 minutes | (payin only; payout equivalent is `ach_cop_bitso`) |

**Note:**

On the payout side, `ach_cop_bitso` (Colombia) settles in around 1 business day, not instantly. High transaction volumes may affect estimated delivery times on any rail.

In development, every payin auto-completes about 30 seconds after initiation, regardless of payment method.

Pix payin quotes with `is_otc: true` (BRL-only, quote expires in 10 seconds; see [Quote expiry windows](#quote-expiry-windows)) also close out on a fixed daily cutoff rather than a rolling window: BlindPay waits for the deposit until 23:59 BRT (America/Sao_Paulo) the same day, or the next day if the quote was created after that time. This cutoff is also the expiration BlindPay sets on the Pix QR code itself.

## Currency minimums

Payin quotes enforce a minimum and maximum `request_amount` (minor units) per currency. Most currencies share the same bounds, but COP's minimum is far higher in raw minor units, reflecting its much smaller nominal value per unit:

| Currency | Minimum | Maximum |
| --- | --- | --- |
| USD | $10.00 | $100,000 |
| BRL | R$10.00 | R$100,000 |
| ARS | $10.00 | $100,000 |
| MXN | $10.00 | $100,000 |
| EUR | €10.00 | €100,000 |
| **COP** | **$2,000.00** | $100,000 |

**Note:**

Thresholds are enforced at quote-creation time and can change; treat the min/max as "varies by currency, the quote enforces it" rather than hardcoding these numbers in your integration. Payin methods without an approved virtual account (`ach`, `wire`) are additionally capped at $500,000 per transaction. `rtp` is capped the same way even when the receiver has an approved virtual account, since RTP always settles through BlindPay's memo-code account rather than the dedicated account.

**Note:**

The Maximum column above is a currency-level ceiling. The maximum that actually applies to a given payin quote is the **receiver's per-transaction limit** (see [KYC limits](kyc.md#limits)), which defaults to $10,000-$50,000 depending on their KYC tier and can be lower than the currency ceiling shown here. The `LIMITS_AMOUNT_OUT_OF_RANGE` error names the exact range that applied to your quote; read it rather than assuming the table above.

**Note:**

These minimums apply only to quotes you create through the API. Deposits into a [virtual account](../virtual-accounts/virtual-accounts.md) skip them entirely: any positive amount forms a payin, including a $0.01 account-verification micro-deposit.

MXN, COP, and ARS payin quotes settle in whole currency units. A sender-denominated quote (`currency_type: "sender"`) with a `request_amount` that isn't a whole unit is rejected with `request_amount_must_be_a_whole_currency_unit`. A receiver-denominated quote (you specify the stablecoin amount and let BlindPay convert) has no such check on input, but the computed sender-side amount is truncated to the nearest whole unit before the deposit is registered, so the payer-facing amount can come out slightly lower than the FX conversion would otherwise produce.

## Quote expiry windows

Every quote has a limited lifetime. Read `expires_at` from the response rather than assuming a fixed TTL, since some rails can return a shorter window.

| Quote type | Default expiry | Notes |
| --- | --- | --- |
| Payin quote | 5 minutes | OTC (BRL-only) payin quotes expire in **10 seconds** instead |
| Payout quote | 5 minutes | May be **shorter** for SEPA payout quotes, since the underlying rail's own deadline can be tighter than 5 minutes |
| Transfer quote | 5 minutes | Fixed; transfers execute immediately after creation |

**Warning:**

`expires_at` is returned in **epoch milliseconds**, not seconds. Divide by 1000 only if your date library expects seconds.

## Onboarding SLAs

| Service | Timeline |
| --- | --- |
| New instance creation | Up to 3 business days |
| KYC Standard | About 60 seconds (automated) |
| KYC Enhanced | 3 hours to 1 business day (manual review) |
| KYB Standard | 3 hours to 1 business day (manual review) |
| Virtual account: compliance review | Part of overall review |
| Virtual account: bank review | Varies by banking partner and account type; typically several business days |
| Limit increase review | Reviewed on submission of supporting documents |

For a limit increase review, the accepted supporting documents are:

- **Individuals:** bank statement, tax return, or proof of income
- **Businesses:** bank statement, financial statements, or tax return

KYC Standard customers are typically auto-approved or auto-rejected after about 60 seconds. If the compliance team needs to review manually, the customer stays in `verifying` until they decide. KYC Enhanced and KYB Standard always require manual review.

Virtual accounts go through a two-stage review: compliance review (`pending_review`) followed by bank review (`verifying`), before reaching `approved` or `rejected`. In development, virtual accounts auto-approve. If the bank requests additional documents during its review, the SLA clock restarts from the date the new documents are submitted.

## Compliance holds

Certain transactions or onboarding steps may be held for compliance review. Common triggers:

- First-time withdrawals or unusual activity patterns
- Large transaction amounts relative to a customer's history
- Sanctions or watchlist screening matches
- Customers from high-risk countries (always routed to Enhanced KYC, manual review)
- Open [requests for information](kyc.md) on a customer (status `compliance_request`, which cannot stack, only one RFI is open at a time)

While a hold is open, the related customer, payin, payout, or virtual account stays in a pending or verifying state until compliance clears it. A payin or payout can also land `on_hold` for manual review after the funds side has already been captured: for payouts, this applies to all USD ACH/Wire/RTP/SWIFT payouts, not only risk-flagged ones. Either can take up to 30 days to resolve: approval resumes the normal flow, and a timeout without a decision fails the transaction.

## Failed or refunded transactions

A payin or payout may fail or be refunded if:

- Beneficiary or bank account details are incorrect
- The receiving bank rejects the payment
- The receiving account is closed or restricted
- Compliance requirements are not met

Refund timing depends on which side of the rail the funds are on:

- **Stablecoin refunds** (the blockchain wallet side) process immediately, since BlindPay is non-custodial and funds simply return to the originating wallet.
- **Fiat refunds** are credited once the funds are returned from the banking network, which depends on that bank's own processing time. Fees may apply to fiat refunds.

**Note:**

In development, force these outcomes on a payin or payout by setting `request_amount` to `66600` ($666.00, forces `failed`) or `77700` ($777.00, forces `refunded`).

## Related

- [Bank transfer in](../payins/payins.md): payin payment methods and arrival windows
- [Pay out to bank](../payouts/payouts.md): payout bank account types and ETAs
- [Payin quotes](../payins/payin-quotes.md): creating and consuming a payin quote before its expiry
- [Virtual accounts](../virtual-accounts/virtual-accounts.md): virtual account review stages and statuses
- [KYC requirements](kyc.md): verification levels, required fields, and limits
