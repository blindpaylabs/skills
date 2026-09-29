# Limit increase

Request higher per-transaction, daily, or monthly transfer limits for a customer by submitting a supporting document, and track the request through compliance review.

Source: https://blindpay.com/docs/learn/limit-increase

Every customer starts with transfer limits set by their KYC tier. When a customer needs to move more than their tier allows, request a limit increase: you submit the new limits you want plus a supporting document, BlindPay's compliance team reviews it, and the approved limits (which may differ from what you asked for) take effect on approval.

## Default limits

Transfer limits are calculated on the stablecoin amount transferred. Each customer has separate limits for payouts (sending) and payins (receiving).

|                 | KYC Standard | KYB Standard | KYC Enhanced |
| --------------- | ------------ | ------------ | ------------ |
| Per transaction | US$ 10,000   | US$ 30,000   | US$ 50,000   |
| Daily           | US$ 50,000   | US$ 100,000  | US$ 100,000  |
| Monthly         | US$ 100,000  | US$ 250,000  | US$ 500,000  |

## Prerequisites

**Before you start:**

1. Create an account at https://app.blindpay.com/sign-up
2. Create a development instance (see essentials/instances.md)
3. Create your API key (see essentials/api-keys.md)

The customer must exist and have `kyc_status` of `approved`; you cannot request a limit increase for a customer still in KYC review. You also need the supporting documents already hosted at URLs, for example via the [Upload](upload.md) endpoint.

Only one request can be in flight per customer at a time. If the customer already has a request with `status: "in_review"`, a new request returns `400 customer_already_has_limit_increase_in_review`. Wait for the existing request to be approved or rejected before submitting another.

Each requested amount (`per_transaction`, `daily`, `monthly`) is capped at 100,000,000,000 (US$ 1,000,000,000.00) in USD cents. A value above that is rejected.

## Supporting document types

Pick the document `type` that matches the customer type and the document you are submitting:

| Customer type | Accepted values |
| --- | --- |
| Individual | `individual_bank_statement`, `individual_tax_return`, `individual_proof_of_income`, `individual_pay_stub`, `individual_employment_letter`, `individual_investment_statement`, `individual_crypto_exchange_statement`, `individual_blockchain_wallet_statement` |
| Business | `business_bank_statement`, `business_financial_statements`, `business_tax_return`, `business_contract`, `business_accounts_receivable`, `business_merchant_processor_statement`, `business_shareholder_loan` |

See the [knowledge base source of funds guide](../kb/source-of-funds.md) for what compliance expects each document to show.

## Request a limit increase

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID, `re_000000000000` with your customer ID.

Amounts are in USD cents: `100000` means US$ 1,000.00.

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/limit-increase \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "per_transaction": 100000,
    "daily": 200000,
    "monthly": 1000000,
    "supporting_documents": [
      { "type": "individual_bank_statement", "file": "https://example.com/statement.pdf" },
      { "type": "individual_tax_return", "file": "https://example.com/tax-return.pdf" }
    ]
  }'
```

You can attach up to 5 documents in `supporting_documents`. For a single document you can still send the shorthand `supporting_document_type` and `supporting_document_file` fields instead.

The response returns the request's ID:

```json
{
  "id": "rl_000000000000"
}
```

## Track the request

The same path lists every limit increase request for the customer, newest state included:

```bash [cURL]
curl --request GET \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/limit-increase \
  --header 'Authorization: Bearer YOUR_API_KEY'
```

Each entry carries the requested amounts, the review status, and, once reviewed, the approved amounts:

| Field | Meaning |
| --- | --- |
| `status` | `in_review`, `approved`, or `rejected` |
| `per_transaction`, `daily`, `monthly` | The amounts you requested, in USD cents |
| `approved_per_transaction`, `approved_daily`, `approved_monthly` | What compliance actually granted; can be lower than requested. `null` until reviewed |
| `supporting_documents` | Every document submitted with the request, as `{ type, file }` |
| `supporting_document_type`, `supporting_document_file` | The first document, kept for backward compatibility |

**Note:**

Compliance can approve lower limits than requested. Always read the `approved_*` fields rather than assuming the requested amounts were granted.

## Testing

On a development instance, every limit increase request is auto-approved immediately at the exact amounts requested: `status` is `approved` as soon as the request is created, `reviewed_by` is `sandbox-auto-approval`, and `approved_per_transaction`, `approved_daily`, `approved_monthly` mirror what you asked for exactly. There is no compliance review to wait on, so you see the full `limitIncrease.new` -> `limitIncrease.update` flow without needing a real reviewer. On a production instance, a request stays `in_review` until BlindPay's compliance team decides, and the approved amounts can be lower than requested.

## Webhooks

| Event | Fires when |
| --- | --- |
| `limitIncrease.new` | A limit increase request is created |
| `limitIncrease.update` | Compliance approves or rejects the request (immediately, on a development instance) |

See [Webhooks](webhooks.md) for signature verification and delivery details.

## Related

- [Customers](customers.md): KYC tiers and the default limits
- [Upload](upload.md): host the supporting document
- [Knowledge base: source of funds](../kb/source-of-funds.md): what each supporting document must show
