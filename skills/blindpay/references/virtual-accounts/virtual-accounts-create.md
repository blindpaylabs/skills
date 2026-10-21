# Create a virtual account

Create a virtual account with the BlindPay API, from customer prerequisites and required compliance fields to the webhooks that confirm approval.

Source: https://blindpay.com/docs/virtual-accounts-create

The request is one API call. The work is in the setup. The customer needs approved KYC, the compliance fields the banking partner checks, and a linked wallet for settlement. This page goes through each step in order and ends with the webhooks that tell you the account is live.

## Prerequisites

**Before you start:**

1. Create an account at https://app.blindpay.com/sign-up
2. Create a development instance (see essentials/instances.md)
3. Create your API key (see essentials/api-keys.md)

You must also [create a customer](../getting-started/quickstart-payin.md) and add a [blockchain wallet](../payins/blockchain-wallets.md) before generating a virtual account. The customer's `kyc_status` must already be `approved`: you cannot create a virtual account for a customer still in KYC review. The wallet is where stablecoin settlement lands behind the scenes; see [Blockchain wallets](../payins/blockchain-wallets.md) for the full mechanics.

## Required fields

### For businesses

Make sure the following fields are filled in on the customer before requesting a virtual account:

- `account_purpose`, `business_type`, `business_description`, `business_industry` (NAICS code), `estimated_annual_revenue`, `source_of_wealth`, `publicly_traded`
- At least one registered business owner, with `ownership_percentage` and `title`
- Owners living in the US must have their SSN in `tax_id`

If you set `account_purpose` to `other`, `account_purpose_other` becomes required (up to 512 characters describing the purpose). This isn't specific to virtual accounts: it's enforced the same way on customer create and update whenever `account_purpose` is submitted.

Depending on the banking partner, the API rejects the creation request with a `missing_required_fields` error naming each blank field, so fill these in before calling the endpoint rather than relying on manual review to catch them.

**Note:**

You can find the `business_industry` NAICS code list at [api.blindpay.com/reference](https://api.blindpay.com/reference#tag/available/GET/v1/available/naics).

To update these fields on an existing customer:

```bash [cURL]
curl --request PUT \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000 \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "email": "john@doe.com",
    "tax_id": "000000000",
    "country": "US",
    "owners": [
        {
            "ownership_percentage": 50,
            "title": "CTO",
            "id": "ub_000000000000"
        }
    ],
    "account_purpose": "business_expenses",
    "business_type": "corporation",
    "business_description": "description",
    "business_industry": "541511",
    "estimated_annual_revenue": "0_99999",
    "source_of_wealth": "business_dividends_or_profits",
    "publicly_traded": false
}'
```

### For individuals (sole proprietors)

Individual customers need `account_purpose` and `source_of_wealth` filled in on the customer, subject to the same `missing_required_fields` rejection as businesses.

Sole proprietors must also include a supporting document in the virtual account creation request itself:

- `sole_proprietor_doc_type`: type of supporting document, one of `master_service_agreement`, `salary_slip`, `bank_statement`
- `sole_proprietor_doc_file`: a URL pointing to the uploaded document

## The request

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID, `re_000000000000` with your customer ID.

**Note:**

After creation, a production virtual account starts in `pending_review` status. It moves through compliance and bank review before becoming `approved`. You'll get a webhook notification at each transition.

```bash [Business]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/virtual-accounts \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
  "banking_partner": "cfsb",
  "token": "USDC",
  "blockchain_wallet_id": "bw_000000000000",
  "partner_fee_id": "pf_000000000000"
}'
```

```bash [Sole proprietor]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/virtual-accounts \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
  "banking_partner": "cfsb",
  "token": "USDC",
  "blockchain_wallet_id": "bw_000000000000",
  "sole_proprietor_doc_type": "master_service_agreement",
  "sole_proprietor_doc_file": "https://example.com/document.pdf"
}'
```

```bash [Colombia (Bre-B)]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/virtual-accounts \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
  "banking_partner": "bre_b",
  "token": "USDC",
  "blockchain_wallet_id": "bw_000000000000"
}'
```

```bash [Mexico (SPEI CLABE)]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/virtual-accounts \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
  "banking_partner": "spei",
  "token": "USDC",
  "blockchain_wallet_id": "bw_000000000000"
}'
```

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `banking_partner` | `jpmorgan`, `citi`, `hsbc`, `cfsb`, `portage`, `bre_b`, `spei` | Yes | Which value is available depends on your instance and the customer's eligibility, see [Banking partner eligibility](#banking-partner-eligibility) below. |
| `token` | `USDC`, `USDT`, `USDB` | Yes | Settlement stablecoin. `USDT` requires the linked wallet's network to be Polygon, Ethereum, or Solana. `USDB` is development-only. |
| `blockchain_wallet_id` | string (`bw_...`) | Yes | Must belong to the same customer. |
| `sole_proprietor_doc_type` | `master_service_agreement`, `salary_slip`, `bank_statement` | Conditional | Required for individual customers on this banking partner. |
| `sole_proprietor_doc_file` | URL string | Conditional | Required alongside `sole_proprietor_doc_type`. |
| `bre_b.gender` | `male`, `female` | No | Only for `bre_b`. Omit it to use the gender on the customer's KYC ID document. Send it when the verification has none, or the request fails with `bre_b_gender_required`. Not stored on the customer. |
| `partner_fee_id` | string (`pf_...`) | No | Pins a [partner fee](../essentials/partner-fees.md) to this account's automated deposits, overriding the instance-wide virtual accounts default fee. Omit to use the default. |

The response includes `id` (`va_...`), `banking_partner`, `kyc_status`, `token`, `blockchain_wallet_id`, `partner_fee_id`, and a `us` object with `ach`, `wire`, and `rtp` sub-objects (each `{routing_number, account_number}`), plus SWIFT and receiving-bank details once the account is `approved`. Before approval, rail numbers are empty.

A `bre_b` account returns `kyc_status: approved` right away, with `us: null` and a `co` object holding `co.bre_b.key`, the Bre-B key payers send COP to.

A `spei` account also returns `kyc_status: approved` right away, with `us: null` and an `mx` object holding `mx.spei.clabe`, the CLABE payers send MXN to.

You can update `token`, `blockchain_wallet_id`, and `partner_fee_id` on an existing virtual account (`partner_fee_id: null` clears the pin back to the instance default); `banking_partner` cannot be changed after creation.

## Banking partner eligibility

- `cfsb` requires the customer's `country` to be `US`. If the customer is an individual, `business_type` must also be `sole_proprietorship`.
- `citi` requires the customer to be a business, and its `country` to not be `US`.
- `bre_b` requires a Colombian document: individuals need a Colombian ID card (`id_doc_country: CO`), businesses need `country: CO` and a NIT in `tax_id`. Anyone else gets `bre_b_requires_colombian_document`. If the document already has a Bre-B key elsewhere, the request fails with `bre_b_document_already_linked`.
- A business customer requesting `cfsb` or `citi` must also have a `country` on BlindPay's supported list for that banking partner; contact BlindPay if you're unsure whether a given country qualifies.

## Webhooks

| Event | Fires when |
| --- | --- |
| `virtualAccount.new` | A virtual account is successfully created (every status path, including development). |
| `virtualAccount.complete` | The banking partner confirms the account and `kyc_status` flips to `approved`. Rail numbers are populated in the payload. |

See [Webhooks](../essentials/webhooks.md) for signature verification and delivery details.

## Related

- [Virtual accounts](virtual-accounts.md): what it is, account types, and statuses
- [Bank transfer in](../payins/payins.md): how deposits into the account become payins
- [Blockchain wallets](../payins/blockchain-wallets.md): where settlement lands
- [Knowledge base: virtual account requirements](../kb/virtual-accounts.md): source of funds and source of wealth documents
