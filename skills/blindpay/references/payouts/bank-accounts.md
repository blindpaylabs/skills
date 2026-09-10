# Bank accounts

Add recipient bank accounts BlindPay pays out to, across SWIFT, ACH, wire, RTP, Pix, PIX Safe, TED, SPEI, ACH COP, Transfers, and SEPA rails.

Source: https://blindpay.com/docs/bank-accounts

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

A bank account represents the recipient details BlindPay pays out to when you execute a payout. A customer can hold multiple bank accounts, and you can add bank accounts that belong to someone other than the customer.

**Advanced flavor (stablecoin and blockchain mechanics)**

A bank account represents the recipient details BlindPay pays out to when a payout converts a customer's stablecoin balance to fiat. A customer can hold multiple bank accounts, and you can add bank accounts that belong to someone other than the customer.

**Note:**

You can add **third-party bank accounts**: a customer named "John" can have a payout sent to a bank account belonging to "Jack". Set `recipient_relationship` to anything other than `first_party` to skip the name-match check.

## How it works

All bank account data must be valid, even on development instances. Validation (regex, length, country rules) runs the same way regardless of instance type; only the downstream provider submission is skipped in development.

### Supported payout rails

| `type` | Country | Estimated time of arrival |
| --- | --- | --- |
| `international_swift` | Global | ~5 business days |
| `ach` | United States | ~2 business days |
| `wire` | United States | ~1 business day |
| `rtp` | United States | instant |
| `pix` | Brazil | instant |
| `pix_safe` | Brazil | instant |
| `ted` | Brazil | ~1 business day |
| `spei_bitso` | Mexico | instant |
| `ach_cop_bitso` | Colombia | ~1 business day |
| `transfers_bitso` | Argentina | instant |
| `sepa` | Europe (SEPA zone) | ~1 business day |

**Note:**

High transaction volumes may affect estimated payout delivery times.

### Required fields per type

| Type | Required fields | Notes |
| --- | --- | --- |
| `international_swift` | `name`, `account_class`, `recipient_relationship`, `swift_code_bic`, `swift_account_holder_name`, `swift_account_number_iban`, full beneficiary address, full bank address | See International SWIFT rules below. Also picks up the [compliance fields](#compliance-fields-for-ach-wire-rtp-and-swift) below |
| `ach` | `name`, `recipient_relationship`, `beneficiary_name`, `routing_number`, `account_number`, `account_type`, `account_class`, `address_line_1`, `city`, `state_province_region`, `country`, `postal_code` | Blocked for `light` KYC customers. Also picks up the [compliance fields](#compliance-fields-for-ach-wire-rtp-and-swift) below |
| `wire` | Same as `ach` | Blocked for `light` KYC customers. Also picks up the [compliance fields](#compliance-fields-for-ach-wire-rtp-and-swift) below |
| `rtp` | Same as `wire` | `routing_number` must be RTP-eligible. Blocked for `light` KYC customers. Also picks up the [compliance fields](#compliance-fields-for-ach-wire-rtp-and-swift) below |
| `pix` | `name`, `pix_key` | `pix_key` can be a CPF, CNPJ, phone, email, or random key |
| `pix_safe` | `name`, `beneficiary_name`, `pix_safe_cpf_cnpj`, `pix_safe_bank_code`, `pix_safe_branch_code`, `account_number`, `account_type` | Use this instead of `pix` when you have the recipient's bank routing rather than a PIX key. `pix_safe_cpf_cnpj` is an 11-digit CPF or 14-digit CNPJ, `pix_safe_bank_code` is the bank's ISPB code, and `account_number` needs a check-digit suffix (for example `12345-6`) |
| `ted` | `name`, `beneficiary_name`, `ted_cpf_cnpj`, `ted_bank_code`, `ted_branch_code`, `account_number`, `account_type` | Same field shape as `pix_safe`, except `ted_bank_code` is keyed by COMPE instead of ISPB |
| `spei_bitso` | `name`, `spei_protocol`, `spei_clabe`, `beneficiary_name` | `spei_institution_code` required for `debitcard`/`phonenum` protocols |
| `ach_cop_bitso` | `name`, `ach_cop_beneficiary_first_name`, `ach_cop_beneficiary_last_name`, `ach_cop_document_id`, `ach_cop_document_type`, `ach_cop_email`, `ach_cop_bank_code`, `ach_cop_bank_account`, `account_type` | `ach_cop_document_type` is `CC`, `CE`, `NIT`, `PASS`, or `PEP` |
| `transfers_bitso` | `name`, `transfers_type`, `transfers_account`, `beneficiary_name` | `transfers_type` is `CVU`, `CBU`, or `ALIAS` |
| `sepa` | `name`, `account_class`, `sepa_iban`, `sepa_beneficiary_bic`, `sepa_beneficiary_legal_name`, `sepa_beneficiary_address_line_1`, `sepa_beneficiary_city`, `sepa_beneficiary_postal_code`, `sepa_beneficiary_country` | The IBAN's country code must match `sepa_beneficiary_country`. Some destinations are individual-only; see [Payment methods](../kb/payment-methods.md#sepa-destinations) |

`account_type` is `checking` or `saving`. `account_class` is `individual` or `business`. `recipient_relationship` accepts `first_party`, `employee`, `independent_contractor`, `vendor_or_supplier`, `subsidiary_or_affiliate`, `merchant_or_partner`, `customer`, `landlord`, `family`, or `other`.

## Prerequisites

**Before you start:**

1. Create an account at https://app.blindpay.com/sign-up
2. Create a development instance (see essentials/instances.md)
3. Create your API key (see essentials/api-keys.md)

You also need a [customer](../getting-started/overview.md) who has completed KYC.

## Add a bank account

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID, `re_000000000000` with your customer ID.

```bash [🌎 International SWIFT]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/bank-accounts \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
  "type": "international_swift",
  "name": "Display Name",
  "account_class": "business",
  "swift_code_bic": "EXAMPLECHXXX",
  "swift_account_holder_name": "Example Beneficiary GmbH",
  "swift_account_number_iban": "CH4008735681787160333",
  "swift_beneficiary_address_line_1": "75 Example Strasse",
  "swift_beneficiary_country": "CN",
  "swift_beneficiary_city": "ZUG",
  "swift_beneficiary_state_province_region": "ZG",
  "swift_beneficiary_postal_code": "8008",
  "swift_bank_name": "Example Bank, N.A.",
  "swift_bank_address_line_1": "18-20 Example Lane",
  "swift_bank_address_line_2": "PO BOX 3941",
  "swift_bank_country": "CN",
  "swift_bank_city": "GENEVA",
  "swift_bank_state_province_region": "GE",
  "swift_bank_postal_code": "1221",
  "recipient_relationship": "vendor_or_supplier",
  "swift_payment_code": "cn_swift_cgoddr"
}'
```

```bash [🇺🇸 ACH]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/bank-accounts \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
  "type": "ach",
  "name": "Display Name",
  "beneficiary_name": "Jane Doe",
  "routing_number": "121000358",
  "account_number": "3211237578",
  "account_type": "checking",
  "account_class": "individual",
  "address_line_1": "Rua Jose Pena Medina, 150",
  "address_line_2": "Apt 902",
  "city": "Vila Velha",
  "state_province_region": "ES",
  "country": "BR",
  "postal_code": "29101320",
  "recipient_relationship": "first_party",
  "phone_number": "+5527999999999",
  "tax_id": "12345678900"
}'
```

```bash [🇺🇸 Wire]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/bank-accounts \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
  "type": "wire",
  "name": "Display Name",
  "beneficiary_name": "JANE DOE",
  "routing_number": "026073008",
  "account_number": "8211239565",
  "account_class": "individual",
  "address_line_1": "5 Penn Plaza",
  "city": "NY",
  "state_province_region": "NY",
  "country": "US",
  "postal_code": "10001",
  "recipient_relationship": "first_party"
}'
```

```bash [🇺🇸 RTP]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/bank-accounts \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
  "type": "rtp",
  "name": "Display Name",
  "beneficiary_name": "JANE DOE",
  "routing_number": "026073008",
  "account_number": "8211239565",
  "account_class": "individual",
  "address_line_1": "5 Penn Plaza",
  "city": "NY",
  "state_province_region": "NY",
  "country": "US",
  "postal_code": "10001",
  "recipient_relationship": "first_party"
}'
```

```bash [🇧🇷 Pix]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/bank-accounts \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
  "type": "pix",
  "name": "Display Name",
  "pix_key": "<Replace this>"
}'
```

```bash [🇧🇷 PIX Safe]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/bank-accounts \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
  "type": "pix_safe",
  "name": "Display Name",
  "beneficiary_name": "Jane Doe",
  "pix_safe_cpf_cnpj": "12345678900",
  "pix_safe_bank_code": "<Replace this>",
  "pix_safe_branch_code": "0001",
  "account_number": "123456-7",
  "account_type": "checking"
}'
```

```bash [🇧🇷 TED]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/bank-accounts \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
  "type": "ted",
  "name": "Display Name",
  "beneficiary_name": "Jane Doe",
  "ted_cpf_cnpj": "12345678900",
  "ted_bank_code": "<Replace this>",
  "ted_branch_code": "0001",
  "account_number": "123456-7",
  "account_type": "checking"
}'
```

```bash [🇲🇽 SPEI]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/bank-accounts \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
  "type": "spei_bitso",
  "name": "Display Name",
  "beneficiary_name": "<Replace this>",
  "spei_protocol": "<Replace this>",
  "spei_institution_code": "<Replace this>",
  "spei_clabe": "<Replace this>"
}'
```

```bash [🇨🇴 ACH COP]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/bank-accounts \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
  "type": "ach_cop_bitso",
  "name": "Display Name",
  "account_type": "checking",
  "ach_cop_beneficiary_first_name": "<Replace this>",
  "ach_cop_beneficiary_last_name": "<Replace this>",
  "ach_cop_document_id": "<Replace this>",
  "ach_cop_document_type": "<Replace this>",
  "ach_cop_email": "<Replace this>",
  "ach_cop_bank_code": "<Replace this>",
  "ach_cop_bank_account": "<Replace this>"
}'
```

```bash [🇦🇷 Transfers 3.0]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/bank-accounts \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
  "type": "transfers_bitso",
  "name": "Display Name",
  "beneficiary_name": "<Replace this>",
  "transfers_type": "<Replace this>",
  "transfers_account": "<Replace this>"
}'
```

```bash [🇪🇺 SEPA]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/bank-accounts \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
  "type": "sepa",
  "name": "Display Name",
  "account_class": "individual",
  "sepa_iban": "DE89370400440532013000",
  "sepa_beneficiary_bic": "COBADEFFXXX",
  "sepa_beneficiary_legal_name": "<Replace this>",
  "sepa_beneficiary_address_line_1": "<Replace this>",
  "sepa_beneficiary_city": "<Replace this>",
  "sepa_beneficiary_postal_code": "<Replace this>",
  "sepa_beneficiary_country": "DE"
}'
```

Save the bank account ID (`ba_...`) for use in payout quotes.

## Connect with Plaid

**Note:**

This feature is gated by `subscription_features.plaid` on the instance. Contact BlindPay to enable it. Calling the endpoint below without it enabled returns a 400 `plaid_not_supported` error.

Instead of entering ACH details manually, a customer can connect their bank account through Plaid. BlindPay reads the verified routing and account numbers directly from Plaid, so there's no manual entry and no micro-deposit wait. The resulting bank account is `type: "ach"`, carries the timestamp `plaid_connected_at`, and can fund an `ach_pull` payin instead of a manual bank transfer; see [Payins](../payins/payins.md#pull-funding-from-a-plaid-connected-account).

There is a single endpoint. It returns a `url` to a BlindPay hosted page that renders nothing but Plaid Link; render it in an iframe on your own page, or open it in a new tab, and BlindPay does the rest. The `token` in the URL is a signed, short-lived session token that carries everything the page needs; treat the URL as a secret for that customer.

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID, `re_000000000000` with your customer ID.

```bash [Create link]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/bank-accounts/plaid \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
  "redirect_url": "https://example.com/bank-connected"
}'
```

```json [Response]
{
  "url": "https://app.blindpay.com/e/plaid?token=eyJhbGciOiJIUzI1NiJ9..."
}
```

```html [Embed]
<iframe src="URL_FROM_THE_RESPONSE" title="Connect your bank" width="100%" height="640" allow="clipboard-write"></iframe>
```

| Field | Type | Notes |
| --- | --- | --- |
| `redirect_url` | string (URL), optional | Where to send the customer once the account is connected. BlindPay appends `bank_account_ids` (comma separated `ba_...` ids) as a query parameter. Leave it out and the page replaces Plaid Link with a plain "Bank account connected" confirmation and a Close button instead. |

The link is single use and expires after at most 4 hours; request a new one every time the customer opens the flow. Sandbox instances always receive a Plaid sandbox session (use Plaid's `user_good` / `pass_good` test credentials), production instances a Plaid production session.

The bank account is created the moment the customer finishes in Plaid Link, one per account they selected, and the `bankAccount.new` webhook fires for each. If you embed the page in an iframe, it also posts a message to the parent window so you can close the frame without waiting for the redirect:

```json [postMessage payload]
{ "source": "blindpay-plaid-link", "event": "connected", "bank_account_ids": ["ba_000000000000"] }
```

`event` is `connected`, `exit` (the customer closed Plaid Link without connecting) or `error` (with a `message`). Check `event.origin === "https://app.blindpay.com"` before trusting the message.

### Connected accounts are always first party

BlindPay only accepts a connected account that belongs to the customer it is being connected for. Before the bank account is created, BlindPay runs Plaid Identity Match between the account holder on file at the bank and the customer record (legal name, plus email, phone, and address when present). The account holder's legal name has to match the customer's name (first and last name for individuals, legal name for businesses) at Plaid's recommended threshold; otherwise the connection is rejected and no bank account is created. Accounts held by someone else fail with `plaid_account_not_first_party`; accounts where the bank returns no holder identity fail with `plaid_identity_unavailable`.

All the identity fields on the resulting bank account (beneficiary name, address, tax id) come from the customer record, never from the bank connection, and `recipient_relationship` is always `first_party`.

The same connection is never turned into two bank accounts, even if the customer reloads the page or Plaid redelivers its notification.

**Note:**

BlindPay never returns or stores the Plaid access token in plaintext anywhere reachable from the API, logs, or webhook payloads.

## Compliance fields for ACH, wire, RTP, and SWIFT

`ach`, `wire`, `rtp`, and `international_swift` bank accounts all pick up the same three country-conditional compliance fields. `pix`, `pix_safe`, `ted`, `spei_bitso`, `ach_cop_bitso`, `transfers_bitso`, and `sepa` never require them.

**Business accounts**

- `business_industry` (NAICS code) is required when `account_class` is `business`.

**Phone number**

- `phone_number` is required when the beneficiary's country (`country` on `ach`/`wire`/`rtp`, `swift_beneficiary_country` on `international_swift`) is one of: BR, CN, CO, HK, MY, MX, PH, UG, UY.

**Tax ID**

- `tax_id` is required when the beneficiary's country (same field as above) is one of: AR, BY, BR, CL, CN, CO, CR, EC, GT, HN, JP, KZ, KR, MX, PK, PE, PH, RU, TH, UY. It's the local tax ID for that country (for example, CPF for Brazil, SSN for the US).

## International SWIFT rules

International SWIFT accounts also have these extra fields, on top of the compliance fields above.

**SWIFT payment code**

`swift_payment_code` is required when `swift_bank_country` is one of: AE, BH, CH, CN, HK, ID, IN, JP, KE, PH, PK, ZA. Omitting it for one of these countries fails bank account creation; leaving it unresolved on an account that already exists fails the payout instead.

**India (IFSC)**

`swift_ifsc_branch_code` is required when `swift_bank_country` is `IN`, in the format `AAAA0NNNNNN`: a 4-letter bank code, then `0`, then a 6-character branch code (for example `HDFC0001234`). BlindPay uppercases it automatically.

**Intermediary bank (optional)**

When the payment needs to route through a correspondent bank, add:

- `swift_intermediary_bank_swift_code_bic`
- `swift_intermediary_bank_account_number_iban`
- `swift_intermediary_bank_name`
- `swift_intermediary_bank_country`

All four are optional and independent of each other; omit them if the beneficiary's bank has a direct correspondent relationship.

**Format constraints**

- `swift_code_bic` must be 8 or 11 characters, uppercase, in standard SWIFT/BIC format.
- `swift_beneficiary_address_line_1` and `_line_2` combined must be 70 characters or fewer; the full beneficiary address (both lines, city, state/province, postal code, country) must be 140 characters or fewer combined.
- `swift_beneficiary_state_province_region` is exactly 2 alphanumeric characters (some countries use a numeric ISO 3166-2 code here instead of letters), and `swift_beneficiary_postal_code` is capped at 16 alphanumeric characters. Both are uppercased automatically.
- `swift_account_holder_name` has no length cap; `swift_bank_name` is capped at 80 characters.

**Note:**

For international SWIFT payouts, compliance documents are collected after the payout is created, not at quote creation time. The payout is placed `on_hold` until the required documents are submitted and approved.

## Look up available rails and fields

These endpoints are public: no API key required for `/available/rails` and `/available/bank-details`.

**List payout rails.** `GET /available/rails` returns the same rail list as the [Supported payout rails](#supported-payout-rails) table above, live: an array of `{ label, value, country }`. `country` is a display hint for the rail picker's flag icon, not a claim that BlindPay has a rail in that country (SEPA, for example, surfaces `DE` as its default flag).

**Fetch a rail's required fields.** Instead of hardcoding the [Required fields per type](#required-fields-per-type) table above, fetch it at runtime:

```bash [cURL]
curl --request GET \
  --url 'https://api.blindpay.com/v1/available/bank-details?rail=ach'
```

Each item in the response array is a field descriptor:

| Property | Type | Meaning |
| --- | --- | --- |
| `key` | string | The bank account field this maps to |
| `label` | string | Display label |
| `regex` | string | Validation pattern, empty string if unconstrained |
| `required` | boolean | Whether the field is always required |
| `items` | array | Picklist options (`{label, value, is_active}`), present when the field is a picklist |
| `requiredWhen` | object | `{field, operator, values}`: the field becomes required only when another field's value matches |

`requiredWhen.operator` is one of `in`, `eq`, `notIn`, or `notEq`.

**Note:**

`requiredWhen` is guidance for your form, not enforcement: this endpoint never validates across fields. The actual requiredness is enforced when you create the bank account, so send the field whenever `requiredWhen` matches or the create call rejects it.

For `spei_bitso` and `ach_cop_bitso`, the bank-code field's `items` picklist is fetched live from BlindPay's banking partner, and an inactive bank still appears in it flagged `is_active: false` rather than being filtered out server-side; filter those out yourself. For `pix_safe` and `ted`, it comes from BlindPay's own bundled Brazilian bank list instead, and is always active. Every other rail's bank-code field has no picklist at all.

**Look up a bank by SWIFT/BIC code.** `GET /available/swift/{swift}` looks up a bank's name, city, branch, and country for a given SWIFT/BIC code, useful for prefilling `swift_bank_name` and the bank address fields before the customer confirms them. `swift` must be exactly 8 or 11 uppercase alphanumeric characters (for example `BOFAUS3NLMA`); anything else fails with a 400 `VALIDATION_FAILED` error before the lookup runs. An 11-character code ending in `XXX` (a bank's head-office suffix) is looked up by its 8-character base code instead; `CHASSGSGXXX` is queried as `CHASSGSG`.

```bash [Request]
curl --request GET \
  --url https://api.blindpay.com/v1/available/swift/BOFAUS3NLMA \
  --header 'Authorization: Bearer YOUR_API_KEY'
```

```json [Response]
[
  {
    "id": "416",
    "bank": "BANK OF AMERICA, N.A.",
    "city": "NEW JERSEY",
    "branch": "LENDING SERVICES AND OPERATIONS (LSOP)",
    "swiftCode": "BOFAUS3NLMA",
    "swiftCodeLink": "https://bank.codes/swift-code/united-states/bofaus3nlma/",
    "country": "United States",
    "countrySlug": "united-states"
  }
]
```

A code with no match returns `200` with an empty array `[]`, not a `404`. If the upstream lookup service errors, BlindPay returns a 400 `retrieve_swift_code_failed` with the upstream error attached, or a 500 `swift_code_api_fetch_failed` for anything unexpected.

## Response fields

The response includes the bank account `id` (`ba_...`), `type`, and the fields you submitted. `account_number` is masked in responses, showing only the last 4 digits.

## Related

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

- [Payouts](payouts.md): create a payout quote and execute a payout to this bank account
- [Payout quotes](payout-quotes.md): lock the rate and fee before paying out
- [Virtual accounts](../virtual-accounts/virtual-accounts.md): a customer's dedicated deposit account
- [Cut-off times](../kb/cut-off-times.md): settlement windows by payment rail
- [Webhooks](../essentials/webhooks.md): `bankAccount.new` fires on create

**Advanced flavor (stablecoin and blockchain mechanics)**

- [Payouts](payouts.md): create a payout quote and execute a payout to this bank account
- [Payout quotes](payout-quotes.md): lock the rate and fee before paying out
- [Managed wallet](../wallets/store.md): the stablecoin balance payouts pull from
- [Cut-off times](../kb/cut-off-times.md): settlement windows by payment rail
- [Webhooks](../essentials/webhooks.md): `bankAccount.new` fires on create
