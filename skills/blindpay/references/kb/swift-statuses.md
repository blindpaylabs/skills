# SWIFT statuses

How SWIFT payout compliance documents are tracked through on-hold, review, and approval via the tracking_documents field.

Source: https://blindpay.com/docs/kb/swift-statuses

## Summary

All international SWIFT payouts require compliance document submission before funds are processed. A payout stays on hold until documents are submitted and approved by BlindPay's compliance team. The `tracking_documents` field on payout responses and webhooks reports progress through `waiting_documents`, `compliance_reviewing`, and approval. Documents must be submitted within 30 days of payout creation and reviewed within 8 days of submission.

## Payout Flow for SWIFT

When you create a [SWIFT payout](../payouts/payouts.md), the flow is:

1. **Payout created** → Status is `on_hold`
2. **Waiting for documents** → `tracking_documents.status: waiting_documents`
3. **Documents submitted** → `tracking_documents.status: compliance_reviewing`
4. **Compliance approved** → Payout proceeds to `processing`, fiat is sent
5. **Compliance rejected** → Status returns to `waiting_documents`, submit new documents

## Submitting Documents

To submit compliance documents and see the full request body, use the document submission endpoint documented on the [Payouts](../payouts/payouts.md) page.

The same endpoint also accepts an optional document on non-SWIFT payouts that are `processing` or `completed`, without any hold or review step. See [Optional documents on other rails](../payouts/payout-quotes.md#optional-documents-on-other-rails).

## Tracking Documents Field

All payout responses and [webhooks](../essentials/webhooks.md) include `tracking_documents`:

```json
{
  "id": "po_xxxxxxxxxxxxx",
  "status": "on_hold",
  "tracking_documents": {
    "step": "processing",
    "status": "waiting_documents",
    "reviewed_by": null,
    "completed_at": null
  }
}
```

### Status Values

| Status                 | Description                           |
| ---------------------- | ------------------------------------- |
| `waiting_documents`    | Awaiting document submission          |
| `compliance_reviewing` | Documents submitted, under review     |
| `null`                 | Documents approved, payout processing |

### Step Values

| Step         | Description                       |
| ------------ | --------------------------------- |
| `processing` | Document verification in progress |
| `completed`  | Document verification complete    |

## Webhook Examples

### Documents Submitted

```json
{
  "id": "po_xxxxxxxxxxxxx",
  "status": "on_hold",
  "tracking_documents": {
    "step": "processing",
    "status": "compliance_reviewing",
    "reviewed_by": null,
    "completed_at": null
  }
}
```

### Compliance Approved

```json
{
  "id": "po_xxxxxxxxxxxxx",
  "status": "processing",
  "tracking_documents": {
    "step": "completed",
    "status": null,
    "reviewed_by": "compliance@blindpay.com",
    "completed_at": "2026-01-27T14:30:00.000Z"
  }
}
```

### Compliance Rejected

```json
{
  "id": "po_xxxxxxxxxxxxx",
  "status": "on_hold",
  "tracking_documents": {
    "step": "processing",
    "status": "waiting_documents",
    "reviewed_by": "compliance@blindpay.com",
    "completed_at": null
  }
}
```

## Timeouts

| Event               | Timeout                         |
| ------------------- | ------------------------------- |
| Document submission | 30 days from payout creation    |
| Compliance review   | 8 days from document submission |

## Accepted Documents

- Invoice
- Contract
- Purchase Order
- Delivery Slip
- Customs Declaration
- Bill of Lading
- Others

Document templates are available in the [SWIFT Deliverability](swift-deliverability.md) guide.

**Warning:**

**Important**: Documents must show the relationship between the sender and the customer. If the document doesn't clearly establish this relationship, the payment will be rejected.

## First Party Payouts

If you are sending funds to your own bank account (the sender and the customer are the same person or entity), set the bank account's `recipient_relationship` to `first_party`. No compliance document is required, so `transaction_document_type`, `transaction_document_id` and `transaction_document_file` do not apply. The beneficiary name must match the customer. This is common when consolidating funds across your own accounts internationally.

## Related

- [SWIFT Deliverability](swift-deliverability.md) · [Payouts](../payouts/payouts.md)
- [Webhooks](../essentials/webhooks.md) · [On-Hold Transactions](on-hold-transactions.md)
