# Customers

People or businesses that send or receive payments and stablecoins through BlindPay.

Source: https://blindpay.com/docs/learn/customers

## What it is

A customer is an individual or business entity designated to interact with BlindPay. You can attach multiple bank accounts, blockchain wallets, and [BlindPay-managed wallets](../wallets/wallets.md) to a customer.

## How it works

For compliance and regulatory requirements, **every customer on your platform must be registered as a customer in BlindPay**. This is mandatory for transaction tracking and reporting. If any of your customers operate as money transmitters (entities that transfer funds on behalf of others), they must also register their end customers as customers in the system. This multi-level registration ensures complete transparency throughout the payment chain.

Every customer must complete a KYC process to verify their identity before sending or receiving funds.

### Required fields

We collect the following data per customer `type` (business, individual) and `kyc_type` (standard, enhanced). Fields marked with `*` are **optional**.

**Note:**

Existing customers can also carry `kyc_type: light`, a legacy tier BlindPay no longer issues. Creating a customer with `kyc_type: "light"` is always rejected, for both `individual` and `business`; you'll only see `light` on customers migrated from BlindPay's older KYC tiers. It still matters downstream: `light` customers are blocked from ACH, wire, and RTP bank accounts, see [Bank accounts](../payouts/bank-accounts.md).

**KYC/B Standard**

| Individual                                               | Business                                                                                 |
| -------------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| First name                                               | Legal name                                                                               |
| Last name                                                | Tax ID (government id number)                                                            |
| Date of birth                                            | Formation date                                                                           |
| Email                                                    | Email                                                                                    |
| Country                                                  | Country                                                                                  |
| Tax ID (government id number)                            | Doing business as\*                                                                      |
| Phone number                                             | Website\*                                                                                |
| IP Address                                               | IP Address                                                                               |
| Country                                                  | Country                                                                                  |
| Address 1                                                | Address 1                                                                                |
| Address 2\*                                              | Address 2\*                                                                              |
| City                                                     | City                                                                                     |
| State/province/region                                    | State/province/region                                                                    |
| Postal code                                              | Postal code                                                                              |
| ID Document - Country                                    | UBOS + Shareholders above 25% (everything from Standard KYC but no Phone and IP Address) |
| ID Document - Type (passport, id card, drivers license)  | Company Formation Document                                                               |
| ID Document - Front                                      | Proof of Ownership Document                                                              |
| ID Document - Back\*                                     | Proof of Address - Type\*                                                                |
| Proof of Address - Type (utility bill, bank statement)\* | Proof of Address - Document\*                                                            |
| Proof of Address - Document\*                            |                                                                                          |
| Selfie File                                              |                                                                                          |

**KYC Enhanced** — everything from KYC/B Standard, plus:

| Individual                          |
| ----------------------------------- |
| Source of Funds Document Type       |
| Source of Funds Document File       |
| Purpose of Transactions             |
| Purpose of Transactions Explanation |

**Note:**

All customers from [high risk countries](../kb/supported-countries.md#high-risk-countries) must go through Enhanced KYC. Enhanced KYC individuals are manually verified by BlindPay's compliance team — this can take up to 1 business day and may require additional documents.

For document quality and submission guidelines, see [KYC Basics](../kb/kyc-basics.md#document-quality).

**Warning:**

When `country` is `US`, `state_province_region` must be exactly two capital letters (for example `NY`, not `New York`). This applies to the customer and to each business owner. Other countries accept the full region name.

`latitude` and `longitude` are optional fields you can send on customer create (latitude between -90 and 90, longitude between -180 and 180). BlindPay forwards them to its identity and fraud-review providers as extra signal; they don't change which KYC tier is required.

### Brazil-specific tax ID validation

When the customer's country is `BR`:

| Type | Rule |
| --- | --- |
| Individual | `id_doc_country` must be `BR`. `tax_id` must be a valid CPF. If `id_doc_type` is `ID_CARD`, a valid RG is also accepted. |
| Business | `country` must be `BR`. `tax_id` must be a valid CNPJ. |

An invalid value returns `400 invalid_cpf_tax_id`, `400 invalid_cpf_or_rg_tax_id`, or `400 invalid_cnpj_tax_id`. The same checks run on both create and update.

### `additional_info`

`additional_info` is a list of supporting documents used for extended verification, most commonly to activate a Zenus-issued virtual account. Each entry is `{ label, value }`, where `value` is typically a file URL from the [Upload](upload.md) endpoint and `label` is one of:

| Label |
| --- |
| `EIN_LETTER` |
| `KYC_PROVIDER_DOCUMENT` |
| `FLOW_OF_FUNDS` |
| `BANK_STATEMENT` |
| `TAX_RETURN` |
| `AUDITED_FINANCIALS` |
| `OPERATING_AGREEMENT` |
| `BOARD_RESOLUTION` |
| `CERTIFICATE_OF_GOOD_STANDING` |
| `BUSINESS_LICENSE` |
| `MSB_LICENSE` |
| `POWER_OF_ATTORNEY` |
| `TRUST_DEED` |
| `SHAREHOLDER_REGISTRY` |
| `BENEFICIAL_OWNER_DECLARATION` |
| `COMPLIANCE_QUESTIONNAIRE` |
| `PROOF_OF_EMPLOYMENT` |
| `REFERENCE_LETTER` |
| `OTHER` |

**Warning:**

`additional_info` is append-only on update. Existing entries are always kept even if you omit them from the request body; only entries beyond the current count get added. You can't remove or reorder an entry that's already saved through this field.

### Review timeline

| Verification Type | Review Timeline            |
| ----------------- | -------------------------- |
| KYC Standard      | ~60 seconds                |
| KYC Enhanced      | 3 hours to 1 business day  |
| KYB Standard      | 3 hours to 1 business day  |

### Statuses

Every customer has a KYC status indicating the current state of their verification:

- **`verifying`**: KYC is being processed
- **`approved`**: KYC verified and approved
- **`rejected`**: KYC rejected
- **`deprecated`**: a legacy status from before the receivers-to-customers migration; you won't see this on customers created going forward
- **`pending_review`**: an Enhanced KYC customer was approved by BlindPay's identity vendor but is held for manual compliance review before the final decision; the customer can't transact yet
- **`awaiting_contract`**: a legacy status kept only for backward compatibility with older records; not set on new customers
- **`compliance_request`**: compliance team opened a [Request for Information](rfi.md) and is waiting for additional documents or clarifications; the customer is paused
- **`approved_rfi`**: the customer is approved and fully operational, but has an open [Request for Information](rfi.md#approved-with-an-open-rfi) they still need to answer

On creation the status is `verifying`. The update timeline depends on the KYC type:

- **KYC Standard**: after ~60 seconds the status typically becomes `approved` or `rejected` automatically. When our compliance team needs to review manually, it stays `verifying` until they decide.
- **KYC Enhanced & KYB Standard**: always require manual review, so the status stays `verifying` until review completes.

When KYC is rejected, BlindPay returns feedback in the `kyc_warnings` or `fraud_warnings` field explaining what to correct. To retry after a rejection, create a brand new customer with corrected information — you cannot update an existing customer's KYC information.

#### Conditional approval for Brazilian businesses

A Brazilian business whose registered activity code (CNAE) does not match its stated activity can be approved on the condition that the CNAE is corrected. In that case the customer object carries a `cnae_condition` with `status: "pending"`, the `expected_cnaes` codes or prefixes, and the `due_at` deadline, and the customer is fully approved in the meantime. Show the deadline to your customer so they can update the CNAE with their accountant. At `due_at` BlindPay rechecks the public CNPJ record. If it now matches, `cnae_condition.status` becomes `passed`. If it still does not, `kyc_status` becomes `rejected`, a `CNAE_MISMATCH` entry is added to `kyc_warnings`, and a `customer.update` webhook is sent.

### Limits

Transfer limits are calculated on the stablecoin amount transferred. Each customer has separate limits for payouts (sending) and payins (receiving).

|                 | KYC Standard | KYB Standard | KYC Enhanced |
| --------------- | ------------ | ------------ | ------------ |
| Per transaction | US$ 10,000   | US$ 30,000   | US$ 50,000   |
| Daily           | US$ 50,000   | US$ 100,000  | US$ 100,000  |
| Monthly         | US$ 100,000  | US$ 250,000  | US$ 500,000  |

### Check remaining limits

The default limits above are the ceiling. To see how much of that ceiling a customer has left, call:

```bash [cURL]
curl --request GET \
  --url https://api.blindpay.com/v1/instances/in_000000000000/limits/customers/re_000000000000 \
  --header 'Authorization: Bearer YOUR_API_KEY'
```

```json
{
  "success": true,
  "limits": {
    "payin": { "daily": 500000, "monthly": 2000000 },
    "payout": { "daily": 300000, "monthly": 1000000 }
  }
}
```

Each number is what's left, in USD cents, not what's been used: the customer's daily/monthly limit minus their completed volume in that window. Day and month boundaries are UTC calendar day and month, not a rolling 24-hour or 30-day window, and only `completed` payins and payouts count. A value can go negative, for example right after a [limit increase](limit-increase.md) request is approved for less than the customer had already used under the previous, higher limit. Returns `404 customer_not_found` if the customer doesn't exist in this instance.

**Warning:**

**Note**: These limits are set for compliance purposes and can be increased upon submission of additional documentation. See [Limit increase](limit-increase.md) for the full flow.

## Prerequisites

**Before you start:**

1. Create an account at https://app.blindpay.com/sign-up
2. Create a development instance (see essentials/instances.md)
3. Create your API key (see essentials/api-keys.md)
4. Accept the Terms of Service and get your `tos_id` (see essentials/terms-of-service.md)

## Create a customer

You can check the required fields in the [BlindPay API Docs](https://api.blindpay.com/reference#tag/customers/POST/v1/instances/{instance_id}/customers).

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID.

```bash [Standard KYC]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers \
  --header 'Authorization: Bearer YOUR_SECRET_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{
    "tos_id": "<replace_with_your_tos_id>",
    "type": "individual",
    "kyc_type": "standard",
    "email": "email@example.com",
    "tax_id": "12345678",
    "address_line_1": "8 The Green",
    "address_line_2": "#12345",
    "city": "Dover",
    "state_province_region": "DE",
    "country": "US",
    "postal_code": "02050",
    "ip_address": "127.0.0.1",
    "phone_number": "+13022006100",
    "proof_of_address_doc_type": "UTILITY_BILL",
    "proof_of_address_doc_file": "https://pub-4fabf5dd55154f19a0384b16f2b816d9.r2.dev/v4-460px-Get-Proof-of-Address-Step-3-Version-2.jpg.jpeg",
    "first_name": "John",
    "last_name": "Doe",
    "date_of_birth": "1998-01-01T00:00:00Z",
    "id_doc_country": "US",
    "id_doc_type": "PASSPORT",
    "id_doc_front_file": "https://pub-4fabf5dd55154f19a0384b16f2b816d9.r2.dev/1000_F_365165797_VwQbNaD4yjWwQ6y1ENKh1xS0TXauOQvj.jpg",
    "selfie_file": "https://pub-4fabf5dd55154f19a0384b16f2b816d9.r2.dev/selfie.png"
  }'
```

```bash [Enhanced KYC]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers \
  --header 'Authorization: Bearer YOUR_SECRET_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{
    "tos_id": "<replace_with_your_tos_id>",
    "type": "individual",
    "kyc_type": "enhanced",
    "email": "email@example.com",
    "tax_id": "123456788",
    "address_line_1": "8 The Green",
    "address_line_2": "#12345",
    "city": "Dover",
    "state_province_region": "DE",
    "country": "US",
    "postal_code": "02050",
    "ip_address": "127.0.0.1",
    "phone_number": "+13022006100",
    "proof_of_address_doc_type": "UTILITY_BILL",
    "proof_of_address_doc_file": "https://pub-4fabf5dd55154f19a0384b16f2b816d9.r2.dev/v4-460px-Get-Proof-of-Address-Step-3-Version-2.jpg.jpeg",
    "first_name": "John",
    "last_name": "Doe",
    "date_of_birth": "1998-01-01T00:00:00Z",
    "id_doc_country": "US",
    "id_doc_type": "PASSPORT",
    "id_doc_front_file": "https://pub-4fabf5dd55154f19a0384b16f2b816d9.r2.dev/1000_F_365165797_VwQbNaD4yjWwQ6y1ENKh1xS0TXauOQvj.jpg",
    "selfie_file": "https://pub-4fabf5dd55154f19a0384b16f2b816d9.r2.dev/selfie.png",
    "source_of_funds_doc_file": "https://pub-4fabf5dd55154f19a0384b16f2b816d9.r2.dev/source-of-funds.jpg",
    "source_of_funds_doc_type": "business_income",
    "purpose_of_transactions": "business_transactions",
    "purpose_of_transactions_explanation": "I am using the money for my personal expenses."
  }'
```

```bash [Standard KYB]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers \
  --header 'Authorization: Bearer YOUR_SECRET_TOKEN' \
  --header 'Content-Type: application/json' \
  --data '{
    "tos_id": "<replace_with_your_tos_id>",
    "type": "business",
    "kyc_type": "standard",
    "email": "test@blindpay.com",
    "tax_id": "123456",
    "address_line_1": "8 The Green",
    "city": "Dover",
    "state_province_region": "DE",
    "country": "US",
    "postal_code": "19901",
    "phone_number": "+13022006336",
    "proof_of_address_doc_type": "UTILITY_BILL",
    "proof_of_address_doc_file": "https://files.blindpay.com/1767022801827-icon.png",
    "legal_name": "Test Inc.",
    "alternate_name": "Test",
    "formation_date": "2000-01-01T00:00:00.000Z",
    "website": "https://test.com",
    "owners": [
      {
        "role": "beneficial_controlling",
        "title": "CEO",
        "ownership_percentage": 100,
        "first_name": "John",
        "last_name": "Doe",
        "date_of_birth": "2000-01-01T00:00:00.000Z",
        "tax_id": "GC200075",
        "address_line_1": "5th Avenue",
        "city": "Manhattan",
        "state_province_region": "NY",
        "country": "US",
        "postal_code": "",
        "id_doc_country": "US",
        "id_doc_type": "PASSPORT",
        "id_doc_front_file": "https://files.blindpay.com/1767022774904-blindpay-square.svg",
        "proof_of_address_doc_type": "UTILITY_BILL",
        "proof_of_address_doc_file": "https://files.blindpay.com/1767022787021-icon.png"
      }
    ],
    "incorporation_doc_file": "https://files.blindpay.com/1767022793017-icon.png",
    "proof_of_ownership_doc_file": "https://files.blindpay.com/1767022796322-icon.png"
  }'
```

### Testing scenarios

By default all customers created in `development` instances are automatically approved. To test a rejection, use `Fail` as the first name (individuals) or legal name (businesses).

## Retrieve a customer

```bash [cURL]
curl --request GET \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000 \
  --header 'Authorization: Bearer YOUR_API_KEY'
```

The response includes `is_tos_accepted`, a boolean computed by comparing the customer's stored terms-of-service acceptance against the version currently in effect. If BlindPay updates the terms after the customer accepted, this flips to `false` even though the customer didn't do anything differently; see [Accepting a new version](terms-of-service.md#accepting-a-new-version).

**Note:**

`id` and `customer_id` are always the same value on a customer. `customer_id` is the current name; `id` is kept for backward compatibility with integrations built before the receivers-to-customers rename. Use either field, they never diverge.

## List customers

```bash [cURL]
curl --request GET \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers \
  --header 'Authorization: Bearer YOUR_API_KEY'
```

Without any pagination parameter, the response is a plain array of customers. Pass `limit`, `starting_after`, or `ending_before` and the response shape changes to a paginated object instead:

```json
{
  "data": [ /* customers */ ],
  "pagination": { "has_more": false, "next_page": null, "prev_page": null }
}
```

Filter the list with query parameters:

| Parameter | Description |
| --- | --- |
| `customer_name` | Matches against first + last name or legal name. Takes priority over `full_name` if you send both. |
| `full_name` | Same match as `customer_name`, kept for backward compatibility. |
| `status` | Filter by `kyc_status`. |
| `customer_id` | Exact match on a single customer's `id`. |
| `bank_account_id` | Only customers holding this bank account. Returns an empty list if the bank account doesn't belong to the instance. |
| `country` | Exact match on the customer's `country`. |
| `limit`, `starting_after`, `ending_before` | Cursor pagination. Omit all three to get the bare array above. |

## Related

- [Wallets](../wallets/wallets.md) · [Bank Accounts](../payouts/bank-accounts.md) · [Blockchain Wallets](../payins/blockchain-wallets.md)
- [KYC Basics](../kb/kyc-basics.md) · [Requests for Information](rfi.md)
