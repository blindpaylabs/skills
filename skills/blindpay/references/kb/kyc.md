# KYC requirements

Verification levels, required fields, statuses, limits, document quality, terms of service, and RFIs for BlindPay customers.

Source: https://blindpay.com/docs/kb/kyc

Every payment flows through a customer that has completed KYC. BlindPay offers two verification levels for individuals and one level for businesses. A customer is an individual or business entity you register before it can send or receive funds; you can attach multiple bank accounts, blockchain wallets, and [managed wallets](../wallets/store.md) to a customer.

For compliance and regulatory requirements, every customer on your platform must be registered as a customer in BlindPay. This is mandatory for transaction tracking and reporting. If any of your customers operate as money transmitters (entities that transfer funds on behalf of others), they must also register their end customers. This multi-level registration keeps the payment chain transparent end to end.

## Verification levels

| Level | Who it applies to | Review type | Typical time |
| --- | --- | --- | --- |
| KYC Standard | Individuals (standard-risk countries) | Automated | ~60 seconds |
| KYC Enhanced | Individuals (high-risk countries) | Manual | 3 hours to 1 business day |
| KYB Standard | Businesses | Manual | 3 hours to 1 business day |

Set `type` (`individual` or `business`) and `kyc_type` (`standard` or `enhanced`) when you create the customer. Individuals from high-risk countries must use `enhanced`; `standard` is rejected for them. Businesses only have one tier, KYB Standard; there is no enhanced KYB. See the [API reference](https://api.blindpay.com/reference#tag/customers) for the complete field list.

**Note:**

Existing customers can also carry `kyc_type: light`, a legacy tier BlindPay no longer issues. Creating a customer with `kyc_type: "light"` is always rejected, for both `individual` and `business`. `light` customers are blocked from ACH, wire, and RTP bank accounts.

## Required fields

Fields marked optional below are not required to create the customer, but missing them can slow down manual review.

### KYC Standard (individual)

| Field | Required |
| --- | --- |
| First name, last name | Yes |
| Date of birth | Yes |
| Email | Yes |
| Phone number, IP address | Optional |
| Country, address line 1, city, state/province/region, postal code | Yes |
| Address line 2 | Optional |
| Tax ID (government ID number) | Yes |
| ID document: country, type (`PASSPORT`, `ID_CARD`, or `DRIVERS`), front file | Yes |
| ID document: back file | Optional |
| Proof of address: type (`UTILITY_BILL`, `BANK_STATEMENT`, `RENTAL_AGREEMENT`, `TAX_DOCUMENT`, `GOVERNMENT_CORRESPONDENCE`), file | Optional |
| Selfie file | Yes |

### KYC Enhanced (individual)

Everything from KYC Standard, plus mandatory:

- Proof of address: type and file (optional on Standard, required here)
- Source of funds document type and file
- Purpose of transactions (`purpose_of_transactions_explanation` becomes required if you send `other`)
- Selfie file

### KYB Standard (business)

| Field | Required |
| --- | --- |
| Legal name, tax ID, formation date | Yes |
| Email, country, address line 1, city, state/province/region, postal code | Yes |
| Website | Yes |
| Alternate name (doing business as), phone number, IP address, address line 2 | Optional |
| Incorporation document | Yes |
| Proof of ownership document | Yes |
| Proof of address: type, file | Yes |
| Owners (UBOs and controlling persons) | Yes, at least 1 |

Each owner provides the same fields as a Standard KYC individual (excluding phone number and IP address), plus a `role` (`beneficial_controlling`, `beneficial_owner`, or `controlling_person`) and optional `ownership_percentage` and `title`. Unlike the primary individual, where proof of address is optional, owners must provide proof of address (type and file). Owners with `country: "US"` must also send `tax_type` (`SSN` or `ITIN`) matching the format of their `tax_id`.

**Note:**

All customers from high-risk countries must go through Enhanced KYC. See the document quality guidance below before you submit files for either level.

## Customer statuses

Every customer has a `kyc_status` indicating the current state of its verification:

| Status | Meaning |
| --- | --- |
| `verifying` | KYC is being processed. This is the status on creation. |
| `approved` | KYC verified and approved. |
| `rejected` | KYC rejected (final). |
| `deprecated` | A legacy status from before the receivers-to-customers migration. Not set on customers created going forward. |
| `pending_review` | An Enhanced KYC customer was approved by BlindPay's identity vendor but is held for manual compliance review before the final decision. The customer can't transact yet. |
| `awaiting_contract` | A legacy status kept only for backward compatibility with older records. Not set on new customers. |
| `compliance_request` | The compliance team opened a Request for Information (RFI) and is waiting on additional documents or clarification. The customer is paused. See [Requests for information](#requests-for-information). |
| `approved_rfi` | The customer is approved and fully operational, but the compliance team opened an RFI they still need to answer. See [Approved with an open RFI](#approved-with-an-open-rfi). |

The update timeline depends on the KYC type:

- **KYC Standard**: after ~60 seconds the status typically becomes `approved` or `rejected` automatically. When the compliance team needs to review manually, it stays `verifying` until they decide.
- **KYC Enhanced and KYB Standard**: always require manual review, so the status stays `verifying` until review completes.

When KYC is rejected, BlindPay returns feedback in the `kyc_warnings` or `fraud_warnings` field explaining what to correct.

**Warning:**

We recommend creating a brand new customer with the corrected fields rather than trying to fix a `rejected` customer in place.

## Limits

Limits are calculated on the stablecoin amount transferred. Each customer has separate limits for payouts (sending) and payins (receiving).

| | KYC Standard | KYB Standard | KYC Enhanced |
| --- | --- | --- | --- |
| Per transaction | $10,000 | $30,000 | $50,000 |
| Daily | $50,000 | $100,000 | $100,000 |
| Monthly | $100,000 | $250,000 | $500,000 |

**Warning:**

These limits are set for compliance purposes. They can be increased upon submission of additional documentation using the [limit-increase endpoint](https://api.blindpay.com/reference#tag/customers).

## Document quality

High-quality documents avoid delays and follow-up requests.

### Accepted formats

- PDF, JPEG, or PNG (WEBP and HEIC/HEIF photos are also accepted and converted automatically)
- Maximum file size: 5 MB
- The entire document must be visible: no cropped edges

### Photo guidelines

- Photograph the original physical document, not a screen or printed copy
- Ensure all text is clear and readable with no glare
- Use good lighting and keep the image in focus
- Submit both front and back if the document has two sides

### Common rejection causes

- Screenshots of documents
- Photos taken of a screen or monitor
- Blurry, dark, or cropped images
- Printed or photocopied documents

## Terms of service

Customers must accept BlindPay's terms of service before you can create them. The terms can only be accepted by a user accessing `https://app.blindpay.com` on the client side; requests from servers are ignored.

**Before you start:**

1. Create an account at https://app.blindpay.com/sign-up
2. Create a development instance (see essentials/instances.md)
3. Create your API key (see essentials/api-keys.md)

### Generate a terms of service URL

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID.

**Note:**

The API only accepts a `uuid` on the `idempotency_key` field. Reusing the same key on a second request is rejected.

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/e/instances/in_000000000000/tos \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "idempotency_key": "<your_uuid>"
  }'
```

The response is a URL with the following query parameters:

```bash [URL example]
https://app.blindpay.com/e/terms-of-service?session_token=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...&idempotency_key=5d8b149e-a55d-4b5b-a8f8-7c4fa315f854&redirect_url=
```

| Param | Required | Example |
| --- | --- | --- |
| `session_token` | Yes | JWT |
| `idempotency_key` | Yes | uuid |
| `redirect_url` | No | `https://yourapp.com/` |
| `customer_id` | No | `re_000000000000` (required when accepting a new TOS version) |

We strongly recommend adding a `redirect_url` parameter so the customer lands back in your application after accepting.

### Accept the terms of service

Open the generated URL for the customer to accept on `https://app.blindpay.com`. After acceptance, BlindPay redirects to your `redirect_url` and appends a `tos_id` query parameter. Copy that `tos_id`: you'll pass it as `tos_id` when you create the customer.

![Terms of service acceptance screen showing where to copy the tos_id](https://pub-4fabf5dd55154f19a0384b16f2b816d9.r2.dev/blindpay_tos_acceptance-min.jpg)

You also receive a `tos.accept` webhook event when the terms of service is accepted.

### Versioning

Each `tos_id` is tied to the terms of service version that was active when it was generated, and it can only ever be linked to one customer. If BlindPay updates the terms of service, calls to the payout quote or payin quote endpoints return an error with message `please_accept_terms_of_service` for customers whose acceptance predates the new version. Generate a new terms of service URL, set `customer_id` on it, and have the customer accept again. Once accepted, the quote endpoints stop returning the error.

## Requests for information

A Request for Information (RFI) is how the compliance team asks for missing or clarifying details when a customer's KYC or KYB review is incomplete. Instead of rejecting the application, BlindPay pauses it, attaches a list of fields the customer needs to fill in, and notifies you so you can collect the response. This is what the `compliance_request` status (above) means.

While a customer is in `compliance_request`, payouts and payins cannot be created for them. There can only be one open RFI per customer at a time; if compliance needs another round, a new RFI is created after the previous one is reviewed.

### Approved with an open RFI

Compliance can also open an RFI without pausing the customer. In that case the customer's `kyc_status` becomes `approved_rfi` instead of `compliance_request`: the customer stays fully operational (payouts, payins, and virtual accounts keep working) while the RFI is open. This is typically used for periodic reviews or follow-up questions on customers that are already approved.

The lifecycle differs from the blocking RFI in three ways:

| | `compliance_request` | `approved_rfi` |
| --- | --- | --- |
| While the RFI is open | Payouts and payins are blocked | Customer keeps full access |
| After the response is submitted | `verifying`, then compliance re-reviews | `approved` immediately, no re-review |
| If the 27-day deadline passes | Customer is auto-rejected | Customer is **not** auto-rejected; compliance reviews the case manually |

The fetch and submit endpoints below work the same for both types, and `customer.update` webhooks fire on every transition, so no integration change is needed to support `approved_rfi`.

### Deadline

When an RFI is opened, the customer has 27 days to respond. If no submission arrives within that window, BlindPay automatically rejects the customer. The deadline is included as `expires_at` in the RFI payload, and every new RFI starts a fresh window.

### RFI status

| Status | Meaning |
| --- | --- |
| `pending` | The RFI is open and waiting for a response. The customer is in `compliance_request` (or `approved_rfi` for the non-blocking type). |
| `submitted` | The response has been received. The customer is back in `verifying` (or `approved` if they were in `approved_rfi`). |
| `expired` | The 27-day deadline passed without a submission. The customer was auto-rejected (customers in `approved_rfi` are not auto-rejected; compliance reviews them manually). |
| `cancelled` | Compliance cancelled the RFI. The customer was restored to its prior status. |

### Receive the webhook

Subscribe to the [`customer.update` webhook](../essentials/webhooks.md). When a customer enters or leaves `compliance_request`, you receive a payload with the new status:

```json
{
  "webhook_event": "customer.update",
  "id": "re_000000000000",
  "instance_id": "in_000000000000",
  "kyc_status": "compliance_request",
  "first_name": "John",
  "last_name": "Doe"
}
```

### Fetch the open RFI

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID, `re_000000000000` with your customer ID.

```bash [cURL]
curl --request GET \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/rfi \
  --header 'Authorization: Bearer YOUR_API_KEY'
```

Returns the open RFI. If none is open for this customer, the response is `200` with a body of `null`, not a `404`:

```json
null
```

Otherwise:

```json
{
  "id": "rfi_a1b2c3d4e5f6",
  "customer_id": "re_000000000000",
  "instance_id": "in_000000000000",
  "status": "pending",
  "request": [
    {
      "title": "Business description",
      "description": "The business description doesn't match the company's website. Please provide a clarification.",
      "fields": [
        { "key": "business_description", "label": "Business description", "required": true },
        { "key": "business_description_explanation", "label": "Explanation", "required": true }
      ]
    },
    {
      "title": "Proof of address",
      "description": "The proof of address attached doesn't match the address provided. Please upload a new one.",
      "fields": [
        {
          "key": "proof_of_address_doc_type",
          "label": "Proof of address type",
          "required": true,
          "items": [
            { "label": "Utility bill", "value": "UTILITY_BILL" },
            { "label": "Bank statement", "value": "BANK_STATEMENT" }
          ]
        },
        {
          "key": "proof_of_address_doc_file",
          "label": "Proof of address file",
          "required": true,
          "regex": "^https://[^\\s]+$"
        }
      ]
    }
  ],
  "response": {},
  "expires_at": "2026-06-08T13:00:00.000Z",
  "submitted_at": null,
  "created_at": "2026-05-12T13:00:00.000Z"
}
```

The `request` field is an array of sections. Each section has a `title`, a `description` written by compliance, and one or more `fields` the customer must fill in:

| Property | Type | Description |
| --- | --- | --- |
| `key` | string | Unique key within the RFI. Use this key in the response body. |
| `label` | string | Label to show above the input. |
| `required` | boolean | If `true`, the field must be present and non-empty in the response. |
| `regex` | string | Optional. A pattern the response value must match. |
| `items` | array | Optional. If present, the field is a dropdown and the response must be one of the provided values. |
| `multiple` | boolean | Optional. If `true`, the field accepts an array of URLs (multiple file uploads, max 20). |

For any file upload field, use the [upload endpoint](https://api.blindpay.com/reference#tag/upload/POST/v1/upload) to host the file and submit the resulting URL.

### Submit a response

The response is a flat object keyed by each field's `key`. There is no `rfi_id` in the URL because there's only ever one open RFI per customer.

**Warning:**

The submission is single-shot. All required fields must be included in one request; there is no partial save, so a submission missing required fields is rejected with `400`.

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/rfi \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "business_description": "We sell B2B SaaS payroll software to mid-market companies.",
    "business_description_explanation": "Updated description matches our public website.",
    "proof_of_address_doc_type": "UTILITY_BILL",
    "proof_of_address_doc_file": "https://files.blindpay.com/1767022801827-bill.pdf"
  }'
```

```json
{ "success": true }
```

The body is validated dynamically against the stored fields: required fields must be present and non-empty, `regex` is applied as a pattern test, `multiple: true` requires an array of URLs (max 20), and unknown keys are rejected. A validation failure returns `400` with details about the offending key.

Once submitted, `customer.update` fires with `kyc_status: "verifying"` while BlindPay re-reviews. After re-review you receive one of `approved`, `rejected`, or `compliance_request` again if a new RFI was opened, in which case you repeat from receiving the webhook. If the 27-day window elapses without a submission, `customer.update` fires with `kyc_status: "rejected"` for the auto-rejection.

For an `approved_rfi` customer the flow is shorter: once the response is submitted, `customer.update` fires with `kyc_status: "approved"` directly, with no re-review step. Missing the deadline does not auto-reject the customer.

## Uploading documents

Use the [upload endpoint](https://api.blindpay.com/reference#tag/upload/POST/v1/upload) to turn a customer's KYC documents and pictures into the file URLs the customer and RFI endpoints expect. BlindPay encrypts files before sharing them with vendors or saving them in the database.

```bash [cURL]
curl 'https://api.blindpay.com/v1/upload?instance_id=in_000000000000' \
  --request POST \
  --header 'Content-Type: multipart/form-data' \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --form 'bucket=onboarding' \
  --form 'file=your_file.pdf'
```

Use the `file_url` returned by the API to populate document fields when creating a customer or submitting an RFI response.

You can optionally pre-screen a document before submitting it: the [document analysis endpoint](https://api.blindpay.com/reference#tag/upload/POST/v1/upload/analyze) reads a PDF, JPG, or PNG and returns an `approval_rate` of `low`, `medium`, or `high` plus a short reason, so you can prompt for a better file before it reaches the official KYC review. It is a signal, not a decision; the official outcome still comes from the verification flow.

## Gotchas

- **Duplicate tax ID is blocked on production.** On production instances, creating a customer with a `tax_id` that matches an existing active customer (`approved` or `approved_rfi`) in the same instance is rejected. This check does not run on development instances. Use a different tax ID or update the existing approved customer's other attachments (bank accounts, wallets) instead of creating a duplicate.
- **Recommended: create a new customer instead of editing a rejected one.** For a `rejected` customer, we recommend creating a new customer with the corrected fields rather than retrying in place.
- **On development instances, customers auto-approve.** Set the first name (individuals) or legal name (businesses) to `Fail` to force a rejection for testing. You can use placeholder URLs for document fields on development.

## Related

- [Webhooks](../essentials/webhooks.md): subscribe to `customer.update` and `tos.accept`
- [Instances](../essentials/instances.md): development vs. production behavior
- [Sandbox vs. production](../essentials/sandbox-vs-production.md): full testing checklist
- [Cut-off times](cut-off-times.md): KYC and onboarding review SLAs
- [Supported countries](supported-countries.md): high-risk and prohibited country lists
