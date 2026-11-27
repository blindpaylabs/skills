# Brazil onboarding requirements

Restricted activities and KYC and KYB requirements for customers with a Brazilian CPF or CNPJ under Central Bank of Brazil Resolutions No. 520/2025 and No. 521/2025.

Source: https://blindpay.com/docs/kb/brazil-onboarding-requirements

## Summary

BlindPay is restructuring its operations to remain compliant with Brazilian laws and regulations. All customers with a Brazilian CPF or CNPJ must meet the onboarding requirements set by Central Bank of Brazil (BCB) Resolution No. 520/2025.

## Regulatory context

On November 10, 2025, the Central Bank of Brazil (Banco Central do Brasil) issued Resolutions No. 519, 520 and 521, which regulate Virtual Asset Service Providers (PSAVs, or VASPs) under Law No. 14,478/2022 and have been in force since February 2, 2026.

- **Resolution No. 520/2025** sets out how PSAVs are constituted and operate, including the authorization regime and the transition rules for companies already active in the market.
- **Resolution No. 521/2025** brings virtual asset operations into the Brazilian foreign exchange market.

## Who this applies to

These requirements apply only to customers with a Brazilian CPF or CNPJ. Customers from other countries are not affected.

## What changed

Moving forward, two changes apply to customers with a Brazilian CPF or CNPJ:

- [Restricted activities](#restricted-activities) that BlindPay cannot support under Central Bank of Brazil regulation
- Updated documentation requirements for [KYC](#know-your-customer-kyc) and [KYB](#know-your-business-kyb)

## Restricted activities

**Not supported under any circumstances.**

Under Resolutions No. 520/2025 and No. 521/2025, BlindPay cannot provide services to customers that carry out the following activities, directly or indirectly.

| Activity | What it means |
| --- | --- |
| **Providing virtual asset services without authorization** | Buying, selling, exchanging or holding custody of virtual assets on behalf of third parties (for example OTC desks, exchanges, brokers and custodians) without being authorized by the Central Bank of Brazil or having an authorization request in progress. |
| **Moving funds on behalf of third parties** | Collecting or receiving funds that belong to third parties in order to buy, sell, exchange or transfer virtual assets for them, or to make payments and international transfers in their interest. The only exception is an institution authorized to operate in the Brazilian foreign exchange market acting in the interest of its own clients. |
| **Cash operations** | Any operation involving cash, in Brazilian or foreign currency. |

BlindPay reviews the activity actually performed by the customer, not only the activity stated in its registration.

## Know your customer (KYC)

Required information and documents:

| Category | Documents and information |
| --- | --- |
| **Identification** | CPF, full name (and social name, when applicable), date and place of birth, nationality, residential address, phone number and email. |
| **Photo ID** | Official photo ID: RG, CNH, legally accepted professional ID card, or passport. |
| **Proof of address** | Proof of residential address issued within the last 90 days. |
| **Identity verification** | Selfie, liveness check and biometric match, or an equivalent mechanism. |
| **Financial profile** | Occupation, employer or economic activity, declared monthly income and net worth, purpose of the relationship, and expected activity (volume, frequency and counterparties). |
| **Source of wealth** | Income tax return (IRPF) and filing receipt, payslips, pró-labore, DECORE, employment or service contract, or other supporting documents such as a sale contract, deed or inheritance records. |
| **Source of funds** | Bank statements for the last 3 months, showing the account holder's name. |
| **Virtual assets** | Declaration of the purpose and nature of the operations, the virtual assets and self-custody wallets used and, when applicable, reasonable evidence of wallet ownership. |

## Know your business (KYB)

Required information and documents:

| Category | Documents and information |
| --- | --- |
| **Corporate documents** | Updated Articles of Association (Contrato Social); Bylaws (Estatuto Social) and shareholder meeting minutes (Ata de Assembleia), when applicable. |
| **Proof of address** | Proof of the company's address. |
| **Ownership structure** | Corporate structure chart showing the ownership chain up to the ultimate beneficial owners (natural persons). |
| **Partners and representatives** | ID documents of partners and legal representatives; power of attorney, when applicable, and the attorney's ID document. |
| **Source of wealth** | Income tax return (IRPF) or income statement (DRE) and revenue for the last 12 months, signed by the responsible accountant and the company's administrator. |
| **Source of funds** | Bank statements for the last 3 months, showing the account holder's name. |
| **Business profile** | Description of the business model, products and services, main clients and suppliers, countries of operation, purpose of the relationship, source of funds and expected activity. |
| **Accounts and wallets** | Bank accounts and wallets to be used, expected inflows and outflows, and relevant counterparties, when applicable. |

**Related persons.** Each legal representative, attorney, administrator, relevant partner, controller and ultimate beneficial owner must also provide the individual [KYC documents](#know-your-customer-kyc) listed above.

**Document acceptance.** Documents that are expired, illegible, incomplete, inconsistent, or whose authenticity cannot be verified are not accepted. Standalone screenshots or unsupported statements do not replace proper evidence.

**Warning:**

**Important note:** Depending on the risk profile, BlindPay may request additional documents.

## Before submitting new applications

Before onboarding a Brazilian company, run this quick check:

1. **Is the CPF or CNPJ valid and up to date?** If not, update the information before submitting.
2. **Does the company carry out any of the restricted activities?** If yes, the application will be rejected.
3. **Does the declared activity match what the company actually does?** BlindPay reviews the activity actually performed, not only the activity in its registration.

Always provide a detailed and accurate description of the Nature of Business. Extensive details are required for a successful onboarding.

## Onboarding flow updates

The BlindPay dashboard and the hosted [onboarding links](https://blindpay.com/docs/learn/onboarding-links) now collect the information and documents listed above. Customers created through the API can send the same data in these fields:

| Requirement | API fields |
| --- | --- |
| Nationality | `nationality` |
| Occupation and employer | `occupation`, `employer` |
| Monthly income and net worth (USD cents) | `monthly_income_amount`, `net_worth_amount` |
| Purpose of the relationship | `account_purpose` (individuals), `account_purposes` and `account_purpose_other` (businesses) |
| Expected activity | `expected_monthly_volume` and `transaction_frequency` (individuals), `monthly_payment_volume` and `payments_per_month` (businesses), `expected_transaction_countries` |
| Source of funds | `source_of_funds_doc_type`, `source_of_funds_description`, `source_of_funds_doc_file` (bank statements) |
| Source of wealth | `source_of_wealth`, `source_of_wealth_doc_file` |
| Corporate documents | `incorporation_doc_file`, `cnpj_registration_doc_file` |
| Ownership structure | `proof_of_ownership_doc_file` |
| Partners and representatives | `owners[].role` (`partner`, `legal_representative`, `attorney_in_fact`), `owners[].power_of_attorney_doc_file` |
| Business profile | `business_description`, `business_activities`, `client_profiles`, `main_suppliers`, `country_of_incorporation` |

These fields are optional on the API today and will become required for customers with a Brazilian CPF or CNPJ. See [Customers](../essentials/customers.md#onboarding-profile-fields) for the accepted values. Until a customer carries this data, additional documents may be requested through an RFI ([Request for Information](information-requests.md)), sent via API or email.

Questions about a specific rejection? Reach our team at [compliance@blindpay.com](mailto:compliance@blindpay.com).

## Related

- [Prohibited activities](prohibited-activities.md) · [KYC requirements](kyc.md) · [KYB documents](kyb-documents.md) · [Source of funds](source-of-funds.md) · [Rejection reasons](rejection-reasons.md)
