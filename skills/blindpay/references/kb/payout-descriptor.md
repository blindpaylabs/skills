# Payout descriptor

How the sender's name appears on a recipient's bank statement varies by payment method and whether Named Account is enabled.

Source: https://blindpay.com/docs/kb/payout-descriptor

## Summary

A payout descriptor is the sender name shown on the recipient's bank statement. By default it shows BlindPay's name (or "Nvio Pagos" for ACH Colombia and Transfers 3.0 Argentina). ACH, Domestic Wire, and International SWIFT support Named Accounts, which display the customer's own name instead.

## Descriptor by Payment Method

| Payment Method | Scenario | Payout Descriptor |
| --- | --- | --- |
| ACH (U.S.) | No Named Account | BlindPay's name |
| ACH (U.S.) | Named Account enabled | Customer's name |
| Domestic Wire (U.S.) | No Named Account | BlindPay's name |
| Domestic Wire (U.S.) | Named Account enabled | Customer's name |
| International SWIFT | No Named Account | BlindPay's name |
| International SWIFT | Named Account enabled | Customer's name |
| RTP (U.S.) | All transfers | BlindPay's name |
| PIX (Brazil) | All transfers | BlindPay's name |
| SPEI (Mexico) | All transfers | BlindPay's name |
| ACH Colombia | All transfers | Nvio Pagos |
| Transfers 3.0 (Argentina) | All transfers | Nvio Pagos |

## Named Accounts

ACH, Domestic Wire, and International SWIFT support Named Accounts, which display the customer's own name as the payout descriptor. To enable named accounts for a customer:

1. Contact BlindPay and specify which customer you want to enable named accounts for.
2. Allow up to **1 business day** for the request to be processed.

Once enabled, recipients see the customer's name on their bank statements instead of BlindPay's name.

**Note:**

**Note:** Named account requests must be submitted to BlindPay directly and take up to 1 business day to process.

## Reference / memo text

The payout descriptor above is the sender name. Separately, `description` on a payout quote gives the recipient a reference line; it accepts up to 128 characters. See [POBO and COBO](pobo-cobo.md#the-reference-the-supplier-sees) for which part of it actually reaches the beneficiary bank.

## Related

- [Cut-off Times](cut-off-times.md) · [Customers](kyc.md)
