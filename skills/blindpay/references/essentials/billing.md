# Billing

How BlindPay charges for virtual accounts, transactions, and partner fees, and when invoices go out

Source: https://blindpay.com/docs/learn/billing

BlindPay charges per active resource and per transaction. There is no setup fee and no monthly minimum. Development instances are always free.

## Virtual accounts

Each active virtual account costs **$1.50 per month**. This applies to US virtual accounts; the fee is charged on your invoice at the end of the billing cycle, not when the account is created.

| Account type | Cost |
| --- | --- |
| US virtual account | $1.50 / mo per active account |
| Brazil virtual account | Not yet published |

**Note:**

All incoming deposits to a virtual account automatically generate a payin. You are billed for the virtual account itself (if applicable), and the transaction fee for each payin is shown in the payin quote before it executes.

See [virtual accounts](../virtual-accounts/virtual-accounts.md) for how to create and manage virtual accounts.

**Note:**

A virtual account deposit's BlindPay fee is charged one of two ways: deducted immediately from the deposit, or accrued to your end-of-month invoice instead. Deposits under $100 always accrue to the invoice, so the customer receives the small deposit in full rather than losing part of it to a fee. Instances on a charge-on-invoice billing configuration accrue every deposit's fee this way, regardless of amount. Contact BlindPay to change your instance's billing configuration.

## Fee schedule

See the flat and percentage fee BlindPay charges on each payment rail and blockchain network configured for your instance.

```bash [cURL]
curl --url https://api.blindpay.com/v1/instances/in_000000000000/billing/fees \
  --header 'Authorization: Bearer YOUR_API_KEY'
```

Requires the owner or admin role on the instance.

Each payment rail (`ach`, `domestic_wire`, `rtp`, `international_swift`, `pix`, `pix_safe`, `ted`, `ach_colombia`, `transfers_3`, `spei`, `sepa`, `faster_payments`) and blockchain network (`tron`, `ethereum`, `polygon`, `base`, `arbitrum`, `stellar`, `solana`) has its own fee object:

| Field | Meaning |
| --- | --- |
| `payin_flat` | Flat payin fee, in cents |
| `payin_percentage` | Payin fee, in basis points (`50` means 0.50%) |
| `payout_flat` | Flat payout fee, in cents |
| `payout_percentage` | Payout fee, in basis points |

`ted` can be `null` if that rail isn't configured for your instance.

## Transactions

Transaction fees vary by payment method and corridor, so BlindPay does not publish a flat rate card. Instead, every quote (payin, payout, or transfer) discloses the exact fee before you execute, so you always see the full breakdown before committing any funds.

The quote response includes:

| Field | Meaning |
| --- | --- |
| `sender_amount` | The amount the sender needs to send |
| `receiver_amount` | The amount the recipient receives |
| `partner_fee_amount` | The partner fee portion, if you have one configured |
| `flat_fee` | The flat fee portion, if any |
| `billing_fee_amount` | The BlindPay billing fee, charged via invoice at the end of the month |

Because the fee is locked into the quote at creation time, nothing changes between the quote and execution. If the quote expires before you execute, request a new one to get current pricing.

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

For payins and payouts, `cover_fees` determines who absorbs the fee: `false` means your customer's recipient amount is reduced by the fee (the common case), `true` means the fee is added on top of what the sender pays. See [receive](../payins/payins.md) and [send](../payouts/payouts.md) for the full breakdown.

**Advanced flavor (stablecoin and blockchain mechanics)**

For payins, payouts, and transfers, `cover_fees` determines who absorbs the fee: `false` means the recipient amount is reduced by the fee (the common case), `true` means the fee is added on top of what the sender sends. See [payins](../payins/payins.md), [payouts](../payouts/payouts.md), and [send](../wallets/send.md) for the full breakdown.

## Partner fees

You can add your own markup on top of BlindPay's fee using partner fees: a percentage or flat amount you configure per payin and per payout. BlindPay collects it from your customers as transactions happen, and releases the accumulated balance to you on the 1st of the following month, net of your own BlindPay invoice for that cycle.

See [partner fees](partner-fees.md) for how to configure, track, and withdraw partner fee revenue.

## Development instances

Development instances never generate invoices. Virtual accounts, payins, payouts, and transfers on a development instance are free to use while you build and test.

## Invoices

Invoices are generated at the end of each billing cycle and sent to the email address on your BlindPay account. Contact [support@blindpay.com](mailto:support@blindpay.com) to update your billing contact.

**Warning:**

If an invoice on a production instance goes unpaid, BlindPay suspends the instance: every route other than billing rejects requests with `400` and `instance_blocked` until the balance is settled. Pay the outstanding invoice, or contact [support@blindpay.com](mailto:support@blindpay.com), to restore access. Development instances are never suspended since they never generate invoices.

## Related

- [Partner fees](partner-fees.md): add a markup on transactions and withdraw monthly revenue
- [Instances](instances.md): dev vs production capabilities
- [Virtual accounts](../virtual-accounts/virtual-accounts.md): account types, fees, and approval stages
