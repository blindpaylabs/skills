# Sandbox vs. production

The exact behavioral differences between development and production instances, plus a checklist for switching over.

Source: https://blindpay.com/docs/learn/sandbox-vs-production

Development and production instances expose the same API surface: same endpoints, same fields, same response shapes. What differs is how money, KYC, and reviews behave behind the scenes. This page lists every difference so nothing surprises you at go-live.

There's also no separate hostname: every request, on either environment, goes to `https://api.blindpay.com`. Which environment a request hits is determined entirely by which instance your API key belongs to, not by the URL you call.

## KYC

On development instances, every customer is auto-approved regardless of the documents you submit. Use any placeholder URLs for document fields, they are not actually verified.

To test a rejection on development, set the first name (individuals) or legal name (businesses) to `Fail`.

On production:

| Verification type | Review timeline |
| --- | --- |
| KYC Standard | About 60 seconds, automatic |
| KYC Enhanced | 3 hours to 1 business day, manual |
| KYB Standard | 3 hours to 1 business day, manual |

Customers from high-risk countries always require Enhanced KYC. Businesses (KYB) always require manual review. A customer's status starts as `verifying` and moves to `approved`, `rejected`, or `compliance_request` (an open request for information) as review completes. See [KYC](../kb/kyc.md) for required fields, limits, and the RFI flow.

**Note:**

On a rejection, BlindPay returns feedback in `kyc_warnings` or `fraud_warnings`. You cannot update an existing customer's KYC data; create a new customer with corrected information instead.

## Test token

Development instances use **USDB**, a BlindPay-issued test stablecoin, in place of real USDC/USDT. Fund a managed wallet by creating a [payin](../payins/payin-managed-wallet.md) targeting it: on a development instance the payin auto-completes about 30 seconds after creation and delivers USDB into the wallet.

**Note:**

Real USDC or USDT cannot be used on a development instance. Production instances use real USDC/USDT and skip USDB entirely.

## Payout behavior

On development instances, payouts skip the fiat payment rails entirely: there is no real bank transfer. Payouts still go through the same on-chain authorization step, but once you create the payout, the payout status moves to `completed` automatically instead of waiting on a real bank transfer.

**Advanced flavor (stablecoin and blockchain mechanics)**

The authorization step itself is unchanged between development and production: ERC-20 `approve` on EVM chains, a signed XDR on Stellar, or token delegation on Solana.

On production, the payout actually moves fiat to the recipient's bank account, with timing dependent on the bank account `type` (see the table in [Cut-off times](../kb/cut-off-times.md)).

## Payin behavior

On development instances, every payin auto-completes about 30 seconds after you call the create-payin endpoint, regardless of payment method. No real transfer needs to arrive.

On production, BlindPay waits for the real payment to arrive before completing the payin. Arrival windows depend on `payment_method`:

| Payment method | Currency | Arrival window |
| --- | --- | --- |
| `ach` | USD | Up to 5 business days |
| `wire` | USD | Up to 5 business days |
| `pix` | BRL | Up to 5 minutes |
| `spei` | MXN | Up to 10 minutes |
| `transfers` | ARS | Up to 10 minutes |
| `pse` | COP | Up to 10 minutes |

## Payables

Development instances have no provider connection for boleto and PIX, so registering a code never checks it against a real bill. Embed one of these digit sequences anywhere in `boleto_barcode` or `pix_qrcode` to force a specific outcome at registration:

| Sequence in the code | Outcome |
| --- | --- |
| `11111` | Registration fails: `unresolvable_payable_code` |
| `22222` | Registration fails: `payable_already_paid_or_expired` (boleto and PIX only) |
| `33333` | Registration fails: `pix_code_amount_required` (PIX only) |
| `44444` | An arrecadação code resolves as a tax bill instead of a utility bill |

Any other digits resolve successfully. The simulated amount comes from the code itself, the same way it would on production: the last 10 digits of a boleto linha digitável, or the PIX EMV amount field. Use that to test different payable amounts without a real bill.

## Document analysis and extraction

On development instances, `POST /upload/analyze` skips the AI call entirely and always returns:

```json
{
  "approval_rate": "high",
  "description": "This is a development instance, all documents are good."
}
```

`POST /upload/extract` also skips the AI call on development instances and returns a fixed sample invoice every time:

| Field | Value |
| --- | --- |
| `amount` | `425000` (US$ 4,250.00) |
| `currency` | `USD` |
| `due_date` | `2026-09-15` |
| `invoice_number` | `INV-DEV-0001` |
| `to.legal_name` | Northwind Cloud Services Inc (San Francisco, CA) |
| `line_items` | 2 fixed items |
| `payment_options` | `ach` and `wire`, same routing/account numbers |

Both hold regardless of the file or `type` you send. Use them to build and test your document-review and invoice-creation flows without spending on real analysis calls.

## Virtual accounts

On development instances, virtual accounts are auto-approved with no compliance or bank review.

On production, virtual accounts go through a two-stage review:

1. **Compliance review**, status `pending_review`
2. **Bank review**, status `verifying`

The result is `approved` or `rejected`. Neither stage guarantees approval. Issuance SLAs vary by account type; see [Virtual accounts](../virtual-accounts/virtual-accounts.md) for the full table. You receive a webhook when the status changes either way.

## Webhooks

Webhooks behave identically on development and production: same event names, same payload shapes, same signature verification. Use the webhook events dashboard on your development instance to inspect deliveries and replay events while you build, before relying on the same flow in production. See [Webhooks](webhooks.md) for the full event list and signature verification.

## Rate limits

Development instances are capped at roughly 100 requests per minute per instance, across every API key and dashboard session that uses the instance. Exceeding it returns a `429` with `development_instance_rate_limit_exceeded`, a `Retry-After` header with the seconds until the window resets, and `RateLimit-Limit` / `RateLimit-Remaining` / `RateLimit-Reset` headers on every response so your client can pace itself.

Production instances have no per-instance request cap.

## Testing amounts

On development instances, you can force a payin or payout into a specific outcome by using one of these amounts as the `request_amount`:

| Amount | Outcome |
| --- | --- |
| `666.00` | Failed |
| `777.00` | Refunded |

Any other amount completes successfully. These overrides only work on development; production processes the real amount you send.

## Switching to production

### Create a production instance

Production instances are created in the [BlindPay dashboard](https://app.blindpay.com), the same way as development ones. New production instances can take up to 3 business days to provision. See [Instances](instances.md).

### Generate a new API key

API keys are scoped to a single instance. A development key will not authenticate against your production instance. See [API keys](api-keys.md).

### Submit real KYC documents

Production KYC is actually verified. Replace placeholder document URLs with real, valid documents (ID, selfie, proof of address) for every customer you create.

### Use real bank account details

Bank account fields must be valid and reachable on production; there is no auto-approve to mask a typo'd routing number or account number.

### Remove testing amounts from your code

Strip any logic that relies on `666.00` or `777.00` to simulate outcomes. On production these are processed as the real amounts they are.

### Switch from USDB to USDC/USDT

Update `token` in your quote requests from `USDB` to `USDC` or `USDT`, and point `network` at a production chain (`base`, `polygon`, `arbitrum`, `ethereum`, `stellar`, `solana`) instead of its testnet equivalent.

### Re-point webhooks

Create a webhook endpoint on the production instance pointing at your production URL; the development instance's webhook configuration does not carry over.

## Related

- [Instances](instances.md): what an instance is and the dev/prod capability table
- [API keys](api-keys.md): per-instance authentication
- [KYC](../kb/kyc.md): verification levels, required fields, and limits
- [Webhooks](webhooks.md): events, payloads, and signature verification
- [Cut-off times](../kb/cut-off-times.md): production settlement windows by payment method
- [Analyze Document](analyze-document.md): document rating thresholds
- [Payables](../payables/payables.md): registering and paying a boleto, PIX, or invoice bill
