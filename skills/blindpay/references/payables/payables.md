# Payables

Register a bill, an invoice, a boleto, or a PIX code, then pay it with the standard payout flow.

Source: https://blindpay.com/docs/payables

A payable is a bill you register: an invoice, a boleto, or a PIX code. Once it exists, you pay it exactly like any other payout: quote it, then execute the payout.

That is what separates a payable from a bank account payout. A bank account payout has no amount until you quote it, because you decide who gets paid and how much. A payable's amount instead comes from what you registered (line items, taxes, discount), or, for a boleto, from what the rail resolves at quote time.

A payable is its own transaction, prefixed `pb_`, and belongs to a [customer](../getting-started/overview.md).

## What you can register

A payable needs exactly one destination:

| Destination | Field(s) | Payable today? |
| --- | --- | --- |
| Boleto | `boleto_barcode`: 47-digit linha digitável or 44-digit barcode not starting with 8 | Yes |
| Utility / tax bill (arrecadação) | `boleto_barcode`: 48-digit linha digitável or 44-digit barcode starting with 8 | Yes |
| PIX code | `pix_qrcode`: EMV copia e cola payload | Yes |
| US bank account (invoice) | `bank_account`: the same object `POST /customers/{customer_id}/bank-accounts` takes, `type` `ach` or `wire` | Yes |

Utility and tax bills go in the same `boleto_barcode` field; the kind of bill is detected from the code itself and confirmed at registration. Unlike a boleto, an arrecadação code carries its amount in the barcode (it never accrues interest between quote and payment) and has no due date or registered payer.

On top of a destination, a payable can carry a full invoice shape: `from`/`to` parties (`legal_name`, address fields, `tax_id`), `line_items` (`name`, `quantity`, `price`), a `note`, `discount`, and `taxes`. Invoices increasingly look like this: a line-itemized bill with a boleto or PIX code attached, not just a bare code.

Code payables (boleto, arrecadação, PIX) must be in `BRL`. Invoice payables paid to a bank account must be in `USD`. EVM networks only, for now.

## Register a payable

Registration is one `POST /instances/{instance_id}/payables` call whatever the destination. The two shapes have their own pages:

- [Boleto and PIX payables](payable-boleto-pix.md): send the code, the rail resolves the rest.
- [Invoice payables](payable-invoice.md): send the vendor's bank details and line items, optionally prefilled by reading the PDF with AI.

Both can [attach the original document](payable-invoice.md#attach-the-document).

## List payables

```bash [cURL]
curl --url 'https://api.blindpay.com/v1/instances/in_000000000000/payables?status=draft&customer_id=re_000000000000' \
  --header 'Authorization: Bearer YOUR_API_KEY'
```

Filter by `status` or `customer_id`. Pass `limit`, `starting_after`, or `ending_before` for cursor pagination; omit them and you get the full array back instead of a paginated envelope.

## Retrieve a payable

```bash [cURL]
curl --url https://api.blindpay.com/v1/instances/in_000000000000/payables/pb_000000000000 \
  --header 'Authorization: Bearer YOUR_API_KEY'
```

## Delete a payable

A `draft` payable (no payout executing it yet) can be deleted:

```bash [cURL]
curl --request DELETE \
  --url https://api.blindpay.com/v1/instances/in_000000000000/payables/pb_000000000000 \
  --header 'Authorization: Bearer YOUR_API_KEY'
```

The payable moves to the terminal `canceled` status, a `payable.update` event fires, and the same code can be registered again afterwards. Once any payout exists for the payable the delete is rejected with `payable_not_cancelable`.

There is no update endpoint and no separate quote/pay/reprice/receipt endpoint for payables: paying one is a normal payout.

## Amount semantics

`amount` on a payable is derived: the sum of `line_items[].price × line_items[].quantity`, plus `taxes`, minus `discount`, all in cents. That is what the invoice says is owed.

For a boleto, that derived `amount` is not necessarily what gets charged. Registering the barcode stamps the amount the provider resolves right then as a single line item, so a payable created from a code always carries a real `amount`. The provider re-resolves it at quote time, and the number can move: fines, interest, or a discount that only applies on certain days. The quote's `receiver_amount` is what actually gets paid.

**Warning:**

Quote an overdue boleto twice on different days and you can get two different amounts, because interest keeps accruing. Always pay against a fresh quote rather than a cached figure.

Quoting a boleto can also update the payable itself, not just price it: `due_date` and `to` are re-resolved against the same rail call. `due_date` is overwritten whenever the resolved code carries one; `to` is only filled in if it did not already carry a `legal_name`. A `GET` right after quoting can therefore show a different `amount`, `due_date`, or `to` than the one you got at registration.

Boletos also only clear on Brazilian banking days, 06:30 through 18:30 BRT; above R$250,000.00 the same-day window closes at 14:30 BRT instead. Quoting outside those hours or on a closure defers execution to the next banking day. If that deferral would push execution past the boleto's `due_date`, the quote is rejected with `payable_boleto_would_be_overdue`.

PIX and invoice-with-bank-details payables use the declared `amount` as-is; there is no rail-side recalculation for those. A fixed-amount PIX code sets that amount at registration; a PIX code without an embedded amount takes the `line_items` you send and pays exactly that value.

### Minimum

A payable is held to the same minimum as a payout: the bill must be worth at least 10.00 USD on the paying side. Registration converts the amount and rejects anything under it with `LIMITS_AMOUNT_OUT_OF_RANGE`, so a bill that could never be paid never becomes a draft.

The quote enforces the same floor against the rate and fees of the moment, so a bill sitting a few cents above it can still be rejected later if the rate moves.

Quoting a payable also counts against the customer's daily and monthly payout volume limits, the same running total described in [Limit increase](../essentials/limit-increase.md) that bank-account payouts share. A payable quote that would exceed either one fails with `LIMITS_VOLUME_EXCEEDED`.

## Status lifecycle

```
draft ─┬─→ canceled (DELETE)
       └─→ processing ─┬─→ completed
                       └─→ back to draft (attempt failed or was refunded)
```

A payable has four statuses, and they describe the bill, not the payment attempt:

| Status | Meaning |
| --- | --- |
| `draft` | Registered, no payout executing it. Quotable. A failed or refunded attempt returns the payable here. |
| `processing` | A payout is executing it, including while that payout is held for review or being refunded. |
| `completed` | Paid. Terminal, never regresses. |
| `canceled` | A draft was deleted. Terminal; the code can be registered again. |

The attempt's own lifecycle (compliance hold, release, failure reason, refund) lives on the payout, reachable through the payable's `payout_id`. To retry after a failed attempt, just quote the same payable again.

## Dedupe

Registering the same `boleto_barcode` or `pix_qrcode` twice for the same instance is rejected with `duplicate_payable`: each code has exactly one payable. If a payment attempt fails or is refunded, do not register the code again. Quote the same payable again and pay it. The exception is a PIX code without an embedded amount: a printed merchant QR is paid many times by many people, so each registration with its own `line_items` is a new payable and the code can be registered again at any time.

## How to pay one

Paying a payable is the standard payout flow, not a dedicated endpoint: create a quote with `payable_id`, then execute the payout.

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/quotes \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
  "payable_id": "pb_000000000000",
  "network": "sepolia",
  "token": "USDB"
}'
```

Exactly one of `bank_account_id` or `payable_id` is accepted. With `payable_id`, do not send `request_amount` or `currency_type`: the amount comes from the payable (re-resolved from the rail for a boleto). A payable quote always prices with the sender covering fees: the bill's amount is fixed, so BlindPay grosses up the stablecoin amount you're charged instead of reducing what the payable receives. This is not configurable per payable quote.

An invoice payable's owned bank account is checked again at quote time: it must be `type` `ach` or `wire` (else `PAYABLES_BANK_ACCOUNT_INVALID`), `status: approved` (else `BANK_ACCOUNTS_NOT_APPROVED`), and its settlement currency must match the payable's `currency` (else `PAYABLES_BANK_ACCOUNT_INVALID`). An `ach`-rail invoice also requires the receiver's country to be eligible for that rail.

Save the quote's `id`, then execute the payout the normal way:

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/payouts/evm \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
  "quote_id": "qu_000000000000",
  "sender_wallet_address": "YOUR_WALLET_ADDRESS"
}'
```

**Advanced flavor (stablecoin and blockchain mechanics)**

The quote response carries the same `contract` object a bank account quote does, with the ERC-20 address, ABI, and decimal-adjusted amount to approve, when the funding wallet is external. See [Payable with EVM](payable-evm.md) and [Payable with managed wallet](payable-managed-wallet.md) for the full walk-throughs.

Quotes expire 5 minutes after creation, same as any other quote. To reprice a payable, whether the quote expired or the amount changed, just create a new quote for the same `payable_id`.

## The payout behind a payable

Executing a payable creates a regular payout. It shows up in the payouts list and retrieve like any other payout, with `payable_id` set instead of a bank account; the payable's `payout_id` points back at it. Both objects describe the same single payment.

## Webhooks

Payables emit their own events, and the payload is the same shape the payable endpoints return:

- `payable.new`: the payable was registered (status `draft`).
- `payable.update`: the bill changed; a payout claimed it (`processing`), the attempt failed or was refunded (back to `draft`, payable again), or a draft was deleted (`canceled`). A draft-return payload carries the attempt's `payout_id` and `last_attempt_status` (`failed` or `refunded`), so it is never mistaken for a never-attempted payable.
- `payable.complete`: the payable was paid (status `completed`).

The payout emits its normal `payout.new`/`payout.update`/`payout.complete` events, carrying `payable_id`. The mid-flight detail (compliance hold, release, failure reason, refund) arrives only there; the payable stays `processing` through all of it. One paid bill produces both streams: correlate them through `payable_id`/`payout_id` and never count a `payout.complete` and its `payable.complete` as two payments.

## Related

- [Boleto and PIX payables](payable-boleto-pix.md): register a bill from its code
- [Invoice payables](payable-invoice.md): register a bill paid to a US bank account
- [Payouts](../payouts/payouts.md): the flow that actually pays a payable

**Advanced flavor (stablecoin and blockchain mechanics)**

- [Payable with EVM](payable-evm.md): pay from an external wallet
- [Payable with managed wallet](payable-managed-wallet.md): pay from a BlindPay-custodied wallet
