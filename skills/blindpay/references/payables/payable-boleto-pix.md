# Boleto and PIX payables

Register a boleto, a utility or tax bill, or a PIX code as a payable, then pay it with the standard payout flow.

Source: https://blindpay.com/docs/payable-boleto-pix

Boleto, arrecadação (utility and tax bills) and PIX payables are registered from the code printed on the bill, always in `BRL`. The rail resolves the beneficiary and, for a boleto, the current amount, so you send the code and get a real payable back. To pay one afterwards, see [How to pay one](payables.md#how-to-pay-one).

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID, `re_000000000000` with your customer ID.

## Boleto

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/payables \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
  "customer_id": "re_000000000000",
  "currency": "BRL",
  "boleto_barcode": "34191790010104351004791020150008191070026000",
  "due_date": "2026-08-25"
}'
```

```js [index.js]
const response = await fetch(
  'https://api.blindpay.com/v1/instances/in_000000000000/payables',
  {
    method: 'POST',
    headers: {
      'Authorization': 'Bearer YOUR_API_KEY',
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      customer_id: 're_000000000000',
      currency: 'BRL',
      boleto_barcode: '34191790010104351004791020150008191070026000',
      due_date: '2026-08-25',
    }),
  }
)

const payable = await response.json()
```

```json
{
  "id": "pb_000000000000",
  "instance_id": "in_000000000000",
  "customer_id": "re_000000000000",
  "from": null,
  "to": { "legal_name": "ACME ENERGIA LTDA", "tax_id": "12.345.678/0001-90" },
  "currency": "BRL",
  "line_items": [{ "name": "ACME ENERGIA LTDA", "quantity": 1, "price": 26000 }],
  "note": null,
  "discount": 0,
  "taxes": 0,
  "bank_account_id": null,
  "boleto_barcode": "34191790010104351004791020150008191070026000",
  "pix_qrcode": null,
  "due_date": "2026-08-25",
  "scheduled_at": null,
  "status": "draft",
  "amount": 26000,
  "created_at": "2026-08-18T12:00:00.000Z",
  "updated_at": "2026-08-18T12:00:00.000Z"
}
```

The code is resolved at registration, so the beneficiary, due date and current `amount` come back real (as a single auto line item). See [Amount semantics](payables.md#amount-semantics) for what actually gets charged at payment time.

If you send `line_items` alongside `boleto_barcode` or a fixed-amount `pix_qrcode`, they are discarded: this single auto line item, priced from what the rail resolves, always replaces whatever you sent. The one exception is a PIX code without an embedded amount, where your `line_items` set the value (see [PIX](#pix)). Use `note` if you want your own reference text on the payable.

## PIX

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/payables \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
  "customer_id": "re_000000000000",
  "currency": "BRL",
  "pix_qrcode": "00020126580014br.gov.bcb.pix0136..."
}'
```

```js [index.js]
const response = await fetch(
  'https://api.blindpay.com/v1/instances/in_000000000000/payables',
  {
    method: 'POST',
    headers: {
      'Authorization': 'Bearer YOUR_API_KEY',
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      customer_id: 're_000000000000',
      currency: 'BRL',
      pix_qrcode: '00020126580014br.gov.bcb.pix0136...',
    }),
  }
)

const payable = await response.json()
```

The response has the same shape as the boleto one, with `pix_qrcode` set and `boleto_barcode` null (and `line_items` discarded the same way). The beneficiary and amount come from the code; a PIX payable has no due date and no document.

A PIX code with a fixed amount embedded is priced from the code, and any `line_items` you send are discarded. A code that lets the payer choose the amount (most static merchant QR codes) is registered with the amount you send: pass it as `line_items` (plus `taxes` and `discount` if any) and that total is what gets paid. Sending such a code without an amount is rejected with `pix_code_amount_required`. Because these codes are reused by many payers, they are exempt from `duplicate_payable`: register the same code again for each payment.

```json
{
  "customer_id": "re_000000000000",
  "currency": "BRL",
  "pix_qrcode": "00020126580014br.gov.bcb.pix0136...",
  "line_items": [{ "name": "Table 12", "quantity": 1, "price": 8500 }]
}
```

## Attach the document

A boleto or invoice payable can store the original file (PDF, JPG, PNG, WEBP or HEIC, up to 5MB). PIX payables cannot: a PIX code has no document, and `document_file` on one is rejected with `payable_document_not_allowed_for_pix`.

Upload the file first with `POST /upload` using bucket `documents`, then pass the returned `file_url` as `document_file` when registering the payable. The URL must be one your instance uploaded (`payable_document_invalid` otherwise). The document cannot be changed after registration.

`document_file` is returned on every payable read and webhook. To download it, call `POST /presign` with the URL to get a one-hour signed link.

## Related

- [Payables](payables.md): amount semantics, lifecycle, dedupe and webhooks
- [Invoice payables](payable-invoice.md): bills paid to a US bank account
