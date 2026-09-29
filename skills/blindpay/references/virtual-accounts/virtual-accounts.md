# Virtual accounts

Issue a virtual account (virtual bank account) that receives USD bank transfers and settles automatically to stablecoins like USDC or USDT in your customer's wallet.

Source: https://blindpay.com/docs/virtual-accounts

Your customer gets a real US bank account in their own name, with its own routing and account number. Anyone can send money to it like to any other bank account. Each deposit creates a [payin](../payins/payins.md). BlindPay converts the deposit and sends USDC or USDT (USDB on development instances) to the wallet you linked at creation. You do not build a checkout flow or show payment instructions. The account number is the integration.

**Note:**

How BlindPay fees are charged depends on the deposit amount: deposits below $100.00 accrue the fee to your invoice at the end of the billing cycle (`billing_fee_amount` on the payin), while deposits of $100.00 or more are charged at the time of the transaction, deducted from the amount delivered on-chain (`transaction_fee_amount`). See [Fees on deposits](#fees-on-deposits).

## How it works

Each virtual account belongs to one customer and settles to one blockchain wallet. A customer can hold more than one account. Give the routing and account number to the payer. Track each deposit through the payin it creates.

### Available account types

| Type | Payment methods | Countries | SLA | Cost |
| --- | --- | --- | --- | --- |
| US (individuals and businesses) | ACH, Wire, SWIFT | US and foreign | 24 hours | $1.50/mo per account |
| US (businesses, foreign senders) | ACH, RTP, Wire, SWIFT | Foreign only | 3-5 business days | $1.50/mo per account |
| US (businesses, US senders) | ACH, Wire, SWIFT | US only | 3-5 business days | $1.50/mo per account |

**Note:**

Named payouts: when a virtual account is issued to an end user, all payouts originating from that account are sent under the end user's name rather than a pooled or omnibus account name. This ensures the recipient sees the actual payer on incoming transfers and aligns with originator-information requirements for payment-rail compliance. These accounts can also send and collect third-party payments.

### Fees on deposits

Every deposit is processed regardless of size, including $0.01 account-verification micro-deposits, and always delivers at least $0.01 of stablecoin to the linked wallet. Where the BlindPay fee lands depends on the deposit amount:

| Deposit amount | BlindPay fee | Field on the payin |
| --- | --- | --- |
| Below $100.00 | Accrued to your invoice, charged at the end of the billing cycle | `billing_fee_amount` |
| $100.00 or more | Charged at the time of the transaction, deducted from the delivered amount | `transaction_fee_amount` |

Instances configured for end-of-month billing accrue the fee to `billing_fee_amount` for every deposit, regardless of amount.

If a [partner fee](../essentials/partner-fees.md) applies to the account, it is collected from what remains after any transaction-time deduction, capped so the deposit always delivers at least $0.01. On small deposits this can mean partial collection (for example, a $5.00 fee on a $5.00 deposit collects $4.99) or none at all on micro-deposits.

### Approval process

After a virtual account is created via the API, it goes through a two-stage review:

1. **Compliance review.** The virtual account starts in `pending_review` status while BlindPay's compliance team reviews the application.
2. **Bank review.** Once compliance approves, the account moves to `verifying` while BlindPay's banking partner does final approval.

Issuance SLAs vary by account type, see the table above. Approval is not guaranteed at either stage: both BlindPay's compliance team and the banking partner reserve the right to reject an application. On a development instance, virtual accounts skip both stages and are created directly in `approved` status.

**Note:**

On a development instance, `routing_number` and `account_number` are not real bank numbers. `routing_number` is always `110000000`, and `account_number` is a fake 12-digit number derived from the virtual account's `id`, so the same account always gets the same number across requests and across the `ach`, `wire`, and `rtp` rails. No real bank recognizes either value; they exist so you can build and test webhook handling before switching to production. Production accounts get real, bank-issued routing and account numbers instead.

### Statuses

| Status | Description |
| --- | --- |
| `pending_review` | The virtual account has been created and is awaiting compliance review. |
| `verifying` | Compliance approved. The virtual account has been submitted to the bank. |
| `approved` | The bank has approved the virtual account. It is now active, with real routing and account numbers, and the `virtualAccount.complete` webhook has fired. |
| `rejected` | The virtual account was rejected during compliance or bank review. |

**Warning:**

A customer can't have two non-deleted, non-rejected virtual accounts on the same `banking_partner`, with one exception: on `jpmorgan`, a customer can hold multiple virtual accounts at once, since each corresponds to a separate sub-account with BlindPay's banking partner. For every other banking partner, create a new customer, delete the existing account, or wait for a rejection before retrying.

## Frequently asked questions

**What is a stablecoin virtual account?** A dedicated bank account issued in your customer's name whose incoming bank deposits (ACH, wire, or SWIFT) are automatically converted to stablecoins like USDC or USDT and delivered to a linked wallet.

**How do I create a virtual account for USDC?** Create a customer that has completed KYC, link a wallet as the settlement destination, and call the virtual accounts API. See [Create a virtual account](virtual-accounts-create.md) for the full request.

**Can a virtual account receive international payments?** Yes. US virtual accounts accept ACH, wire, and SWIFT transfers from US and foreign senders, depending on the account type.

**How much does a stablecoin virtual account cost?** US virtual accounts cost $1.50 per month per account. Deposits generate payins; BlindPay fees on deposits below $100.00 are charged on your monthly invoice, and deposits of $100.00 or more are charged at the time of the transaction.

## Next

- [Create a virtual account](virtual-accounts-create.md): prerequisites, required fields, the create request, and webhooks.

## Related

- [Bank transfer in](../payins/payins.md)
- [Blockchain wallets](../payins/blockchain-wallets.md)

**Advanced flavor (stablecoin and blockchain mechanics)**

- [Offramp wallets](../payouts/offramp-wallets.md): the mirror of a virtual account, a deposit address whose incoming stablecoin pays out as fiat

- [Knowledge base: virtual account requirements](../kb/virtual-accounts.md)
- [Learn: webhooks](../essentials/webhooks.md)
