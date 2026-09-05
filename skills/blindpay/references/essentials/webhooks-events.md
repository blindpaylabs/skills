# Webhooks events

The full BlindPay webhook event catalog, grouped by domain, with an example payload.

Source: https://blindpay.com/docs/learn/webhooks-events

Every webhook endpoint you register receives events from the catalog below, unless you narrow the `events` array to a subset when creating the endpoint. Events are grouped by the resource domain that triggers them.

**Note:**

`customer.new`, `customer.update`, and `customer.delete` are the canonical customer lifecycle events. The legacy `receiver.*` events were retired with the July 2026 receivers-to-customers sunset and no longer fire.

## Customer

| Event | Fires when |
| --- | --- |
| `customer.new` | A customer is created |
| `customer.update` | A customer is updated, including a KYC status change from BlindPay's compliance review |
| `customer.delete` | A customer is deleted |

## Bank account

| Event | Fires when |
| --- | --- |
| `bankAccount.new` | A bank account is added to a customer |

## Blockchain wallet

| Event | Fires when |
| --- | --- |
| `blockchainWallet.new` | A blockchain wallet is added to a customer |

## Terms of service

| Event | Fires when |
| --- | --- |
| `tos.accept` | A customer accepts the terms of service |

## Payin

| Event | Fires when |
| --- | --- |
| `payin.new` | A payin is created (any payment method, or a deposit into a virtual account) |
| `payin.update` | The payin advances through an intermediate step, such as an arrival check or manual review |
| `payin.complete` | The payin finishes: sent to the destination wallet, refunded, or failed |

## Payout

| Event | Fires when |
| --- | --- |
| `payout.new` | A payout is created and the source funds have been captured |
| `payout.update` | The payout advances through an intermediate step, such as document checks, manual review, the bank send, or reaching `failed` |
| `payout.complete` | The payout finishes successfully: confirmed at the destination, or refunded. A `failed` payout fires `payout.update`, not `payout.complete`. |

## Virtual account

| Event | Fires when |
| --- | --- |
| `virtualAccount.new` | A virtual account is created or requested for a customer |
| `virtualAccount.complete` | The virtual account is approved and an account number is issued |

## Transfer

| Event | Fires when |
| --- | --- |
| `transfer.new` | A stablecoin [transfer](../wallets/send.md) is created |
| `transfer.update` | The transfer advances through an intermediate step (for example, a cross-chain burn confirmation), or fails to confirm |
| `transfer.complete` | The transfer confirms on-chain |

## Wallet

| Event | Fires when |
| --- | --- |
| `wallet.new` | A managed [wallet](../wallets/wallets.md) is created for a customer |
| `wallet.update` | A managed wallet's [yield](https://blindpay.com/docs/yield) status changes |
| `wallet.inbound` | A stablecoin deposit is detected in a managed wallet |

## Limit increase

| Event | Fires when |
| --- | --- |
| `limitIncrease.new` | A customer requests a transaction limit increase |
| `limitIncrease.update` | BlindPay's compliance review approves or rejects the request |

## Payable

| Event | Fires when |
| --- | --- |
| `payable.new` | A payable (bill) is registered as a draft |
| `payable.update` | The bill changes: a payout claims it (`processing`), the attempt fails or is refunded (back to `draft`), or a draft is deleted (`canceled`) |
| `payable.complete` | The payout executing the payable finishes and the bill is paid |

See [Payables](../payables/payables.md#webhooks) for the full payload shape.

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

## Key events for fiat

If you are only working with fiat rails, the events you will see most often are:

- `virtualAccount.complete`: an account is approved and ready to receive deposits
- `payin.complete`: a deposit arrived and settled
- `payout.new` and `payout.complete`: a bank transfer started, then finished, failed, or was refunded
- `bankAccount.new`: a payout destination was added to a customer

See [Payin quotes](../payins/payin-quotes.md) and [Payout quotes](../payouts/payout-quotes.md) for the requests that lead to these events.

**Advanced flavor (stablecoin and blockchain mechanics)**

## Key events for stablecoin

If you are only working with stablecoin rails, the events you will see most often are:

- `wallet.inbound`: stablecoins landed in a managed wallet
- `blockchainWallet.new`: a customer added an external wallet as a payout destination
- `payin.complete` and `payout.complete`: fiat-to-stablecoin and stablecoin-to-fiat legs finished
- `transfer.new` and `transfer.complete`: an on-chain stablecoin transfer started and confirmed

See [Blockchain wallets](../payins/blockchain-wallets.md) and [Wallets](../wallets/wallets.md) for how these destinations are created.

## Example payload

Every payload includes a `webhook_event` field set to the event name, followed by the resource's own fields. This example is a `payin.complete` event:

```json
{
  "webhook_event": "payin.complete",
  "id": "pi_000000000000",
  "status": "completed",
  "customer_id": "re_000000000000",
  "receiver_amount": 10000,
  "sender_amount": 10000,
  "payment_method": "ach",
  "partner_fee": 0,
  "billing_fee_amount": 0,
  "transaction_fee_amount": 250,
  "blindpay_bank_details": {
    "bank_name": "Example Bank, N.A.",
    "account_number": "000000000000",
    "routing_number": "000000000"
  },
  "memo_code": "BP-AB12CD",
  "pix_code": null,
  "clabe": null,
  "tracking_complete": {
    "step": "completed",
    "completed_at": "2026-06-01T12:00:00.000Z"
  },
  "tracking_payment": {
    "step": "completed",
    "completed_at": "2026-06-01T11:58:00.000Z"
  },
  "tracking_transaction": {
    "step": "completed",
    "transaction_hash": "0x0000000000000000000000000000000000000000000000000000000000000",
    "completed_at": "2026-06-01T11:59:00.000Z"
  }
}
```

## Related

- [Webhooks](webhooks.md): create an endpoint and see webhooks in context
- [Webhooks verification](webhooks-verification.md): verify the signature headers on every call
- [Payins](../payins/payins.md): the payin lifecycle behind `payin.*` events
- [Payouts](../payouts/payouts.md): the payout lifecycle behind `payout.*` events
- [Payables](../payables/payables.md): the bill lifecycle behind `payable.*` events
- [Partner fees](partner-fees.md): how partner fee events relate to quotes and transactions
