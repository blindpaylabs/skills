# POBO and COBO

How payment on behalf of and collection on behalf of work at BlindPay, what puts your customer's name on a Wire, and where the compliance line sits.

Source: https://blindpay.com/docs/kb/pobo-cobo

## Summary

POBO (payment on behalf of) and COBO (collection on behalf of) let a platform send and receive international payments for its customers instead of forcing each one to open its own overseas bank account. At BlindPay both are built on two things: a virtual account issued to a specific onboarded customer, and the Named Account setting that puts that customer's name on the payment instead of BlindPay's. Neither works for a party BlindPay cannot see, which is the difference between POBO/COBO and [nested payments](nested-payments.md).

## Definitions

| Term | Direction | What it means |
| --- | --- | --- |
| **POBO** | Outbound | You pay a supplier for an obligation that belongs to your customer. The payee should be able to tell who the payment is really from. |
| **COBO** | Inbound | You collect a payment that economically belongs to your customer. The payer should be able to send to an account that looks like your customer. |

The terms come from corporate treasury, where a parent company pays and collects through one account on behalf of its subsidiaries. A platform on BlindPay is doing the same shape of thing for its customers rather than its own group companies, which changes the compliance requirements but not the mechanics.

## The Compliance Line

POBO and COBO describe moving money for someone else, and BlindPay prohibits doing that for a party it cannot see. The two are compatible only because of who the "someone else" is.

**Supported.** Your customer is onboarded into your instance, has passed KYC or KYB, and holds a virtual account in their own name. BlindPay can see who owns the funds, who is sending or receiving them, and what the payment is for.

**Not supported.** Your customer uses their account to pay or collect for *their* customers, who BlindPay has never onboarded or screened. This is nesting, regardless of what the payment reference says.

**Warning:**

**Important**: A virtual account that holds a third party's money, where that third party is not itself an onboarded BlindPay customer, is a nested structure. If one of your customers needs to run payments for their own client base, the compliant path is their own BlindPay instance. See [Nested payments](nested-payments.md).

## COBO: Collecting Wires

Inbound international Wires require a virtual account. Creating a SWIFT payin without one returns `international_swift_payins_require_virtual_account`.

The virtual account is issued per customer and carries its own beneficiary name, SWIFT BIC and account number, so the party paying your customer sends to details that identify your customer. See [Virtual accounts](../virtual-accounts/virtual-accounts.md) to create one.

When the Wire lands, the payin reports who sent it:

| Field | What it carries |
| --- | --- |
| `sender_name` | Name of the party that sent the Wire |
| `sender_bank_name` | Their bank |
| `sender_account_number` | Their account |
| `transaction_reference` | The reference they attached, useful for matching an invoice |

These arrive on `GET /payins/{id}` and on the `payin.complete` webhook, so your customer's receivables can be reconciled without asking them who paid.

## POBO: Sending Wires

### Whose Name Appears

By default the payout descriptor shows BlindPay's name. The **Named Account** setting replaces it with your customer's name, and it is available on ACH, Domestic Wire and International SWIFT only. It is enabled per customer on request and takes up to 1 business day to process. Full matrix in [Payout descriptor](payout-descriptor.md).

This is the part worth being precise about: your customer's name reaches the payee because the payment is issued against an account titled for that customer, not because BlindPay writes them into a separate ultimate-party field. See [What BlindPay Does Not Do](#what-blindpay-does-not-do) below.

### Minimum and Timing

| Constraint | Value |
| --- | --- |
| Minimum SWIFT payout | 100 USD. Below this the payout is rejected with `swift_minimum_is_100_usd` |
| Cut-off | 10:30 AM ET |
| Estimated settlement | Up to 5 business days |

### Documents

Every SWIFT payout is created `on_hold`. When the destination bank account's `recipient_relationship` is anything other than `first_party`, the payout also requires a compliance document proving the relationship between the sender and the customer, and it waits until that document is approved.

Sending to your customer's own account (`recipient_relationship` set to `first_party`) requires no document. The beneficiary name must match the customer, otherwise the request fails with `beneficiary_name_must_match_customer_for_first_party`.

Accepted document types are `invoice`, `purchase_order`, `delivery_slip`, `contract`, `customs_declaration`, `bill_of_lading` and `others`. Templates and the address formatting rules are in [SWIFT deliverability](swift-deliverability.md); the review lifecycle and its timeouts are in [SWIFT statuses](swift-statuses.md).

Pass the invoice number as `transaction_document_id`.

### The Reference the Supplier Sees

The `description` on the quote is what reaches the beneficiary bank. It maps to MT103 field 70 (`RmtInf` on pacs.008), which is the only non-bank field the beneficiary's bank surfaces as the payment reference. Put the invoice reference here so your customer's supplier can match the payment to their receivable.

**Warning:**

**Important**: the field accepts 128 characters but only the first **30** reach the beneficiary. Keep the reference short and put the invoice number first. `INV-2026-0418` survives; a full sentence does not.

Two related fields exist and neither is beneficiary-facing. Payment purpose memo and instruction fields map to MT103 field 72, which correspondent banks largely drop, so a reference placed there will not appear on the supplier's statement.

## What the Payment Carries Back

Each rail exposes a different reference, and BlindPay returns only the ones the rail actually provides. All of these arrive on `GET /payouts/{id}` and on the `payout.complete` webhook.

| Rail | `provider_uetr` | `provider_imad` | `provider_reference` | `provider_clearing_system` |
| --- | --- | --- | --- | --- |
| International SWIFT | Yes | No | No | `SWIFT` |
| Domestic Wire | Yes | Yes | Sometimes | `FED` or `CHIPS` |
| ACH | No | No | Sometimes | `ACH` |

- `provider_uetr` is the Unique End-to-end Transaction Reference. The network assigns it, so every bank in the chain refers to the same payment by the same identifier. It is populated once the Wire is confirmed and is null while the payment is in flight.
- `provider_imad` is the Fed Input Message Accountability Data, and exists on domestic Fedwire payments only.
- `provider_reference` is a bank-side reference captured from the account booking webhook. It appears on some ACH and domestic Wire payouts only. It is not the formal ACH network trace number and should not be handed to a bank as one.

`provider_uetr`, `provider_imad`, and `provider_reference` are capped at 120 characters; `provider_clearing_system` at 40.

## What BlindPay Does Not Do

Treasury teams who have run POBO through a corporate bank sometimes expect the ISO 20022 ultimate-party fields. BlindPay does not populate `UltmtDbtr` or `UltmtCdtr`. A payment carries two non-bank parties: the account being debited and the beneficiary. Your customer's identity reaches the payee through the account title (Named Account) and the payment reference, not through a separate ultimate-debtor tag.

Two consequences worth planning around:

- Without Named Account enabled for that customer, the payee sees BlindPay as the sender and will reconcile the payment against BlindPay unless your reference tells them otherwise.
- Some countries restrict or prohibit third-party payments outright, and a beneficiary bank can decline a payment that is not in the invoice party's name as a matter of policy. Check the destination before promising a corridor. See [Supported countries](supported-countries.md).

## Related

- [Virtual accounts](virtual-accounts.md) · [Payout descriptor](payout-descriptor.md)
- [SWIFT deliverability](swift-deliverability.md) · [SWIFT statuses](swift-statuses.md)
- [Nested payments](nested-payments.md) · [Cut-off times](cut-off-times.md)
- Product overview: [POBO and COBO over SWIFT](https://blindpay.com/pobo-cobo-swift)
