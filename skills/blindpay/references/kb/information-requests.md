# Information requests

A Request for Information (RFI) is how BlindPay compliance asks for missing KYC or KYB details before a customer can be approved.

Source: https://blindpay.com/docs/kb/information-requests

## Summary

A Request for Information (RFI) is how BlindPay's compliance team asks for missing or clarifying details when a customer's KYC or KYB review is incomplete. Instead of rejecting outright, BlindPay puts the customer on hold in `compliance_request` status, emails your team, and gives you 27 days to collect and submit the requested documents or explanations from your customer. This guide is the operational walkthrough; to handle RFIs programmatically, see the [RFI API documentation](../essentials/rfi.md).

## When an RFI is created

Compliance creates an RFI whenever a customer's submission is missing information, contradicts another document, or falls into a category that requires extra context. Common triggers include:

- A business description that doesn't match the company's website or industry
- A proof of address document that doesn't match the registered address
- An identity document that's expired, blurry, or partially visible
- An ownership structure that needs an updated share register or UBO documentation
- Source of funds or source of wealth that needs supporting evidence

Each RFI is built as a list of sections. Each section has a title, a description explaining what compliance needs, and one or more fields the customer must fill in (text, file upload, or dropdown). A single RFI can mix document uploads with free-text explanations.

## What happens to the customer

When an RFI is created, the customer's `kyc_status` flips from its prior value to `compliance_request`. See [customer statuses](kyc.md#customer-statuses) for the full status list. While in this status:

- The customer cannot send or receive funds
- The customer sees a banner in their dashboard prompting them to submit the requested information
- A 27-day countdown starts. If no response is submitted, the customer is automatically rejected

**Note:**

The customer's previous status is snapshotted when the RFI is created. If your compliance contact cancels the RFI, the customer returns to that prior status instead of staying in `compliance_request`.

## Email notifications

When an RFI is opened, BlindPay emails **every active member of your instance** so your team can coordinate with the customer. We do not email the customer directly, since collecting the documents and uploading them is your responsibility as the partner.

The cadence is:

| Day | Email           | Purpose                                                                     |
| --- | --------------- | --------------------------------------------------------------------------- |
| 0   | Action Required | Lists each section and the fields requested. Includes the deadline.         |
| 7   | Reminder        | Sent only if the RFI is still pending.                                      |
| 17  | Final Notice    | Sent only if the RFI is still pending. Warns of automatic rejection.        |
| 27  | Auto-Rejected   | Sent if the deadline passes without a submission. The customer is rejected. |

All emails come from `compliance@blindpay.com` and link to your dashboard at `app.blindpay.com`.

**Note:**

Emails are sent to every non-deleted user in your instance. To control who receives them, manage your team's membership in **Settings → Members**, or set a dedicated compliance address list in **Settings → Team → Compliance**: once you add addresses there, RFI emails go only to that list instead of every team member. You can also turn customer RFI emails off entirely from the same page; instance-level RFI emails are unaffected and always send.

## Responding to an RFI

1. Open the customer in your BlindPay dashboard. A banner highlights the open RFI.
2. Review each requested section with your customer and collect the documents or explanations.
3. Upload the files and submit the response in a single action.

Once submitted:

- The customer's `kyc_status` returns to `verifying`
- BlindPay's compliance team re-reviews the application along with the new information
- The pending email reminders for this RFI are stopped immediately

**Warning:**

**Submit the entire RFI at once.** The submission is single-shot. All required fields must be filled before you can send the response.

## RFIs for approved customers

Compliance can also request information from a customer **without pausing them**. In that case the customer's `kyc_status` becomes `approved_rfi` instead of `compliance_request`, and:

- The customer keeps sending and receiving funds normally while the RFI is open
- Submitting the response restores them to `approved` immediately, with no re-review
- Missing the 27-day deadline does **not** auto-reject them; our compliance team reviews the case manually instead

This is typically used for periodic reviews or follow-up questions on customers that are already approved. The email cadence and the response flow in your dashboard are the same.

## What happens after submission

After you submit, compliance reviews the response and either approves the customer or requests another round of information. Most RFIs are resolved within a few business days. If the response still doesn't meet our requirements, a new RFI may be opened with the additional questions.

## Deadlines and auto-rejection

If 27 days pass without a submission, BlindPay automatically rejects the customer. The customer's `kyc_status` becomes `rejected` and a final email is sent to your team. This applies to customers in `compliance_request`; customers in `approved_rfi` are not auto-rejected (see above). To restart the KYC process for that customer, contact our compliance team.

## Best practices

- **Keep your member list clean.** Only members who should see compliance correspondence should remain on the instance. Remove former teammates promptly.
- **Coordinate with your customer early.** The 27-day window is firm, and most rejections happen because the customer wasn't reached in time.
- **Match the format compliance asks for.** If a section requires a utility bill, a bank statement won't satisfy it. Re-read each section before uploading.
- **Submit complete responses.** Partial submissions are not accepted; the dashboard validates every required field before letting you send.

## Related

- [RFI API documentation](../essentials/rfi.md) · [Customer statuses](kyc.md#customer-statuses)
- [Basic Customer Information (KYC)](kyc-basics.md) · [Basic Business Information (KYB)](kyb-documents.md)
