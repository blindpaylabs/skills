# SWIFT deliverability

Requirements for compliance documents and beneficiary address formatting that maximize the chance a SWIFT transfer is delivered.

Source: https://blindpay.com/docs/kb/swift-deliverability

## Summary

SWIFT is the global payment network BlindPay uses to send and receive money internationally, processed exclusively through tier 1 banks. To maximize deliverability, every B2B SWIFT payment requires a compliance document proving the relationship between sender and customer, and beneficiary addresses must follow strict formatting rules with a 140-character limit. Documents that fail to establish the sender-customer relationship cause the payment to be rejected. The minimum SWIFT payout is 100 USD on the requested amount, and payouts below it are rejected with `swift_minimum_is_100_usd`.

## Compliance Documentation

Every B2B payment sent through SWIFT requires a transaction document showing the **relationship between the sender and the customer**.

BlindPay accepts the following transaction documents:

- [Invoice](https://pub-4fabf5dd55154f19a0384b16f2b816d9.r2.dev/invoice-template.pdf)
- [Purchase Order](https://pub-4fabf5dd55154f19a0384b16f2b816d9.r2.dev/purchase-order-template.pdf)
- [Delivery Slip](https://pub-4fabf5dd55154f19a0384b16f2b816d9.r2.dev/delivery-slip-template.pdf)
- [Contract](https://pub-4fabf5dd55154f19a0384b16f2b816d9.r2.dev/contract-template.pdf)
- [Customs Declaration](https://pub-4fabf5dd55154f19a0384b16f2b816d9.r2.dev/customs-declaration-template.pdf)
- [Bill of Lading](https://pub-4fabf5dd55154f19a0384b16f2b816d9.r2.dev/bill-of-lading-template.pdf)
- [Others](https://pub-4fabf5dd55154f19a0384b16f2b816d9.r2.dev/invoice-template.pdf)

**Warning:**

**Important**: If the document doesn't show the relationship between the sender and the customer, the payment will be rejected.

To submit transaction documents via the API and track their review status, see the [SWIFT Statuses](swift-statuses.md) guide.

## How to Add a SWIFT Account

1. In your instance, open **Customers** from the sidebar menu and select the customer you want to register the SWIFT account for.
2. Open **Bank Accounts**, then in **Add Bank Account** select the payment method **International Swift**. You can also add [bank accounts via the API](../payouts/bank-accounts.md).
3. Enter the SWIFT account details:
   - Fill in all fields in **UPPERCASE only**
   - Use plain text only — no punctuation (no commas, periods, or special characters)
   - ✅ **Correct**: `JPMORGAN CHASE BANK NA SINGAPORE`
   - ❌ **Incorrect**: `J.P. Morgan Chase Bank, N.A., Singapore.`
4. If the SWIFT code has only 8 characters, append `XXX`:
   - `CHASSGSG` becomes `CHASSGSGXXX`
   - This routes communication directly to the bank's main branch.
5. Click **Create**. The SWIFT account is now linked to that customer.

## Beneficiary Address Requirements

To ensure successful processing of SWIFT payments, addresses must follow strict formatting rules. All address elements combined must not exceed **140 characters**.

### Address Structure

A complete SWIFT-compatible address consists of:

1. **Address Line 1** - Street name, building number, or primary physical location
2. **Address Line 2** - Additional location details (office, apartment, room, floor)
3. **City** - Name of the city or municipality
4. **State / Province Code** - ISO-style two-character state or region abbreviation
5. **Postal Code** - National postal identifier
6. **Country Code** - ISO 3166-1 alpha-2 country code

### Field Requirements

| Field           | Requirement                                                                                                                          | Example                   |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------ | ------------------------- |
| Address Line 1  | Max 70 characters combined with Addr2. Main street/building only. No P.O. Boxes unless the receiving bank explicitly allows them.     | `123 Gran Via Building A` |
| Address Line 2  | Max 70 characters combined with Addr1. Secondary details only (unit, suite, floor, block).                                           | `Suite 405`               |
| City            | Max 35 characters. Full city name, not abbreviations unless the official name is abbreviated.                                        | `Madrid`                  |
| State / Province Code | Exactly 2 characters. If the country has no states/provinces, repeat the 2-letter country code.                               | US → `CA`, ES → `ES`      |
| Postal Code     | Max 16 characters. All-numeric or alphanumeric. No special symbols. HK → `999077`, UAE → `00000`.                                    | US `94105`, UK `M4A1B3`   |
| Country Code    | Exactly 2 letters, ISO 3166-1 alpha-2.                                                                                               | `ES`                      |

### Example: Correctly Formatted Address

```
Addr1: 123 Gran Via Bldg A
Addr2: Suite 405
City: Madrid
State: ES
Postal Code: 28013
Country: ES
```

**Full combined output** (must be ≤ 140 chars):
`123 Gran Via Bldg A Suite 405 Madrid ES 28013 ES` (59 characters)

### Example: Converting a Long Address

**Original address** (too long for SWIFT):

```
Building 12, Zone C, Longhua Science & Technology Industrial Park,
Minzhi Street, Longhua New District, Shenzhen City, Guangdong Province, China 518131
```

**Converted SWIFT-compatible version**:

```
Addr1: Bldg 12 Zone C Longhua Sci-Tech Ind Park
Addr2: Minzhi St
City: Shenzhen
State: CN (China does not use state codes → use country code)
Postal Code: 518131
Country: CN
```

**Full combined output**:
`Bldg 12 Zone C Longhua Sci-Tech Ind Park Minzhi St Shenzhen CN 518131 CN` (95 characters)

### Address Submission Checklist

Before submitting an address, verify:

- [ ] Addr1 + Addr2 ≤ 70 characters
- [ ] City ≤ 35 characters
- [ ] State is exactly 2 letters
- [ ] Postal code ≤ 16 alphanumeric characters
- [ ] Country is exactly 2 letters
- [ ] Entire address ≤ 140 characters
- [ ] Address contains no unsupported symbols

## Related

- [SWIFT Statuses](swift-statuses.md) · [Bank Accounts](../payouts/bank-accounts.md)
- [Payouts](../payouts/payouts.md)
