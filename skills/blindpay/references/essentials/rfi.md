# RFI

Respond programmatically when BlindPay's compliance team needs additional documentation from a customer.

Source: https://blindpay.com/docs/learn/rfi

## What it is

A Request for Information (RFI) is how BlindPay's compliance team asks for missing or clarifying details when a customer's KYC or KYB review is incomplete. Instead of rejecting the application, BlindPay pauses it, attaches a list of fields the customer needs to fill in, and notifies you so you can collect the response from your customer.

This page covers the **API integration**. For a non-technical walkthrough, see the [Requests for Information guide](../kb/information-requests.md).

## How it works

1. **Create a customer.** Their `kyc_status` starts as `verifying`.
2. **Listen for the `customer.update` webhook.** When compliance opens an RFI, you receive an event with `kyc_status: compliance_request`.
3. **`GET /v1/.../rfi`** to fetch the open RFI and the list of fields the customer needs to fill in.
4. **Collect the fields** from your customer through your own UI.
5. **`POST /v1/.../rfi`** to submit the response in a single shot.
6. **Listen for `customer.update` again.** The `kyc_status` flips back to `verifying` while BlindPay re-reviews.
7. **Wait for the final webhook.** The customer ends up `approved`, `rejected`, or back in `compliance_request` if compliance needs another round.

While the customer is in `compliance_request`, **payouts and payins cannot be created** for them.

### KYC status

The RFI statuses extend the set documented on the [Customers](customers.md#statuses) page:

- **`verifying`**: Initial review or post-RFI re-review
- **`approved`**: KYC has been verified
- **`rejected`**: KYC has been rejected (final)
- **`compliance_request`**: An RFI is open and the customer is paused waiting for your response
- **`approved_rfi`**: An RFI is open but the customer stays approved and fully operational. See [Approved with an open RFI](#approved-with-an-open-rfi).

### Approved with an open RFI

Compliance can open an RFI without pausing an already-approved customer. The customer's `kyc_status` becomes `approved_rfi` and they keep full access (payouts, payins, and virtual accounts keep working) while the RFI is open. The integration is identical: the same webhook, fetch, and submit steps below apply. The differences:

| | `compliance_request` | `approved_rfi` |
| --- | --- | --- |
| While the RFI is open | Payouts and payins are blocked | Customer keeps full access |
| After the response is submitted | `verifying`, then compliance re-reviews | `approved` immediately, no re-review |
| If the deadline passes (27 days, or 5 business days for `approved_rfi`) | Customer is auto-rejected | Customer is **not** auto-rejected; compliance reviews the case manually |

### Deadline

When an RFI is opened on a customer in `compliance_request`, the customer has **27 days** to respond; an RFI opened on an `approved_rfi` customer is due in **5 business days**. If no submission arrives within that window, BlindPay automatically rejects the customer (except customers in `approved_rfi`, who stay operational and are reviewed manually by compliance). The deadline is included as `expires_at` in the RFI payload, and every new RFI starts a fresh window.

**Note:**

There can only be **one open RFI per customer** at a time. If compliance needs another round, a new RFI is created after the previous one is reviewed.

### RFI status

The `status` field on an RFI object reflects its lifecycle:

| Status        | Meaning                                                                              |
| ------------- | ------------------------------------------------------------------------------------ |
| `pending`     | The RFI is open and waiting for a response. The customer is in `compliance_request` or `approved_rfi`. |
| `submitted`   | The response has been received. The customer is back in `verifying` (or `approved` if they were in `approved_rfi`). |
| `expired`     | The 27-day deadline passed without a submission. The customer was auto-rejected (`approved_rfi` customers are not auto-rejected). |
| `cancelled`   | Compliance cancelled the RFI. The customer was restored to its prior status.         |

### Endpoints

| Method | Path                                                                  | Description                              |
| ------ | --------------------------------------------------------------------- | ---------------------------------------- |
| `GET`  | `/v1/instances/{instance_id}/customers/{customer_id}/rfi`             | Fetch the open (pending) RFI, or `null` if none is open.  |
| `POST` | `/v1/instances/{instance_id}/customers/{customer_id}/rfi`             | Submit the response as a flat object.    |

## Prerequisites

**Before you start:**

1. Create an account at https://app.blindpay.com/sign-up
2. Create a development instance (see essentials/instances.md)
3. Create your API key (see essentials/api-keys.md)

## Receive the webhook

Subscribe to the [`customer.update` webhook](webhooks.md). When a customer enters or leaves `compliance_request`, you receive a payload with the new status:

```json
{
  "webhook_event": "customer.update",
  "id": "re_000000000000",
  "instance_id": "in_000000000000",
  "kyc_status": "compliance_request",
  "first_name": "John",
  "last_name": "Doe",
  "...": "..."
}
```

### Who receives RFI emails

By default, every active team member on the instance also gets a customer RFI notification email. Two settings on the [instance](instances.md) change this, both updatable via `PUT /instances/{instance_id}`:

| Setting | Effect |
| --- | --- |
| `compliance_emails` | Up to 20 addresses. When set, customer RFI emails go only to these addresses instead of every team member. |
| `customer_rfi_emails_enabled` | Set to `false` to stop customer RFI emails entirely. Omit on update to leave it unchanged. |

See [Requests for Information guide](../kb/information-requests.md#email-notifications) for the full email cadence.

## Fetch the open RFI

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID, `re_000000000000` with your customer ID.

```bash
curl --request GET \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/rfi \
  --header 'Authorization: Bearer YOUR_SECRET_TOKEN'
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
      "title": "Business Description",
      "description": "The business description doesn't match the company's website. Please provide a clarification.",
      "fields": [
        { "key": "business_description", "label": "Business Description", "required": true },
        { "key": "business_description_explanation", "label": "Explanation", "required": true }
      ]
    },
    {
      "title": "Proof of Address",
      "description": "The Proof of Address attached doesn't match the address provided. Please upload a new one.",
      "fields": [
        {
          "key": "proof_of_address_doc_type",
          "label": "Proof of Address Type",
          "required": true,
          "items": [
            { "label": "Utility Bill", "value": "UTILITY_BILL" },
            { "label": "Bank Statement", "value": "BANK_STATEMENT" }
          ]
        },
        {
          "key": "proof_of_address_doc_file",
          "label": "Proof of Address File",
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

### Request schema

The `request` field is an array of **sections**. Each section is a self-contained question with a `title`, a `description` written by compliance, and one or more **fields** the customer must fill in.

| Property              | Type      | Description                                                                  |
| --------------------- | --------- | ---------------------------------------------------------------------------- |
| `title`               | `string`  | Section heading shown above the inputs.                                      |
| `description`         | `string`  | Compliance's prompt to the customer. Render verbatim.                        |
| `supporting_document` | `string`  | Optional. URL to a template or example document (e.g. a Google Drive link).  |
| `fields`              | `Field[]` | The inputs to render and collect.                                            |

Each entry in `fields`:

| Property   | Type                                  | Description                                                                                          |
| ---------- | ------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `key`      | `string`                              | Unique key within the RFI. **This is the key you must use in the response body.**                    |
| `label`    | `string`                              | The label to show above the input.                                                                   |
| `required` | `boolean`                             | If `true`, the field must be present and non-empty in the response.                                  |
| `regex`    | `string`                              | Optional. A regex pattern that the response value must match.                                        |
| `items`    | `{ label: string, value: string }[]`  | Optional. If present, the field is a dropdown and the response must be one of the provided `value`s. |
| `multiple` | `boolean`                             | Optional. If `true`, the field accepts an **array of URLs** (multiple file uploads).                 |

For any file upload field, use the [Upload endpoint](upload.md) to host the file and submit the resulting URL.

## Submit a response

The response is a **flat object** keyed by `field.key`. There is no `rfi_id` in the URL because there's only ever one open RFI per customer.

**Warning:**

**The submission is single-shot.** All required fields must be included in one request. There is no partial save, so a submission missing required fields is rejected with `400`.

```bash
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/rfi \
  --header 'Authorization: Bearer YOUR_SECRET_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{
    "business_description": "We sell B2B SaaS payroll software to mid-market companies.",
    "business_description_explanation": "Updated description matches our public website.",
    "proof_of_address_doc_type": "UTILITY_BILL",
    "proof_of_address_doc_file": "https://files.blindpay.com/1767022801827-bill.pdf"
  }'
```

Response:

```json
{ "success": true }
```

### Validation

The body is validated dynamically against the stored `request.fields[]`:

- `required` fields must be present and non-empty
- `regex` is applied to the value as a `RegExp` test
- `multiple: true` requires a `string[]` of URLs (max 20)
- Unknown keys (not declared in the request) are rejected

A validation failure returns `400` with details about the offending key.

### After submission

Once submitted, `customer.update` fires with `kyc_status: "verifying"` while BlindPay re-reviews. After re-review you receive one of:

- `kyc_status: "approved"` if the customer passed
- `kyc_status: "rejected"` if the customer failed
- `kyc_status: "compliance_request"` if a new RFI was opened. Repeat from [Receive the webhook](#receive-the-webhook).

If the 27-day window elapses without a submission, `customer.update` fires with `kyc_status: "rejected"` for the auto-rejection.

For a customer in `approved_rfi`, the submission restores them directly: `customer.update` fires with `kyc_status: "approved"` and there is no re-review step. Missing the deadline does not auto-reject them.

## Related

- [Customers](customers.md#statuses) · [Webhooks](webhooks.md) · [Upload](upload.md)
- [Requests for Information guide](../kb/information-requests.md)
