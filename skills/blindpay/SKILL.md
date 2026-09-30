---
name: blindpay
description: BlindPay integration for payins/payouts and stablecoin on/off-ramps (ACH, wire, SWIFT, Pix, SPEI, SEPA, USDC/USDT), customer KYC/KYB, bank and virtual accounts, managed or blockchain wallets, cross-chain transfers, payables (boletos, PIX QR codes, invoices), partner fees, or webhooks.
license: MIT
metadata:
  author: blindpay
  version: "2.0"
  tags: [payments, stablecoin, crypto, fiat, payout, payin, payables, usdc, usdt, virtual-accounts, webhooks]
---

# BlindPay Integration

BlindPay is a global payment API. One integration covers bank rails (ACH, wire, RTP, SWIFT, Pix, SPEI, ACH COP, Transfers 3.0, SEPA) and stablecoins (USDC, USDT) across seven chains. Every payment settles through stablecoins under the hood, and BlindPay is a non-custodial processor: funds stay under the customer's control, and failed transactions are returned to the originating wallet or bank account.

The docs come in two flavors and the reference files keep both, marked inline:

- **Abstracted flavor**: bank-rails framing. Virtual accounts, payins as deposits, payouts as bank transfers. BlindPay handles the stablecoin leg.
- **Advanced flavor**: stablecoin framing. Managed and external wallets, chains and tokens, on-chain authorization, cross-chain transfers.

Both run on the same API, keys, instances and webhooks. When answering, pick the flavor that matches what the user is building.

## Terminology

`receiver` was renamed to **customer**. The API path is `/customers`, IDs still start with `re_`. The legacy `/receivers` path and `receiver.*` webhook events were retired with the July 2026 receivers-to-customers sunset; use `/customers` and `customer.*` events exclusively.

## Authentication

Every call needs a secret API key and an instance ID in the path:

```bash
curl --request GET \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers \
  --header 'Authorization: Bearer YOUR_API_KEY'
```

Keys are per instance. A **development** instance is the sandbox (testnets, simulated fiat, and USDB in place of USDC/USDT — see Chains and tokens below); a **production** instance moves real money. See [Sandbox vs production](references/essentials/sandbox-vs-production.md).

## The two core flows

Both flows are the same **quote-then-execute** pattern — the term BlindPay's own migration docs use throughout: create a quote, execute it before it expires, then track the result over webhooks.

### Payout: money out to a bank account

1. Accept Terms of Service → `tos_id`
2. Create a customer (KYC/KYB) → `re_...`, wait for `approved`
3. Add a bank account → `ba_...`
4. Create a payout quote → `qu_...`, single-use — see "quote expired" under Common errors; the destination is a `bank_account_id`, or a `payable_id` to pay a registered bill (boleto, PIX QR code or invoice — see [Payables](references/payables/payables.md))
5. Fund the payout: a managed wallet balance needs no signature; an external wallet needs authorization (ERC-20 `approve` on EVM, signed XDR on Stellar, token delegation on Solana)
6. Execute the payout → `po_...`, then track it over webhooks

Start at [Payouts](references/payouts/payouts.md).

### Payin: money in from a bank account

1. Accept Terms of Service → `tos_id`
2. Create a customer (KYC/KYB) → `re_...`, wait for `approved`
3. Pick a destination: managed wallet (`bl_...`), external blockchain wallet (`bw_...`), or issue a virtual account for standing deposits
4. Create a payin quote → `pq_...` — see "quote expired" under Common errors
5. Execute the payin → `pi_...` and show the payer the returned instructions (Pix code, CLABE, memo code, ACH/wire details)
6. Funds arrive, BlindPay delivers the stablecoin leg and fires webhooks

Start at [Payins](references/payins/payins.md).

## Main endpoints

All paths are prefixed with `https://api.blindpay.com/v1/instances/{instance_id}`.

| Resource | Endpoint |
| --- | --- |
| Customers | `/customers`, `/customers/{re}` |
| Terms of Service | `/e/instances/{instance_id}/tos` (public, no key) |
| Bank accounts | `/customers/{re}/bank-accounts` |
| Offramp wallets | `/customers/{re}/bank-accounts/{ba}/offramp-wallets` |
| Blockchain wallets | `/customers/{re}/blockchain-wallets`, `.../sign-message` |
| Managed wallets | `/customers/{re}/wallets`, `.../wallets/{bl}/balance` |
| Virtual accounts | `/customers/{re}/virtual-accounts` |
| Payout quotes | `/quotes` |
| Payouts | `/payouts/evm`, `/payouts/stellar`, `/payouts/solana`, `/payouts/stellar/authorize` |
| Payables | `/payables`, `/payables/{pb}` (paid via the payout flow: quote with `payable_id`) |
| Payin quotes | `/payin-quotes` |
| Payins | `/payins/evm` |
| Transfers | `/transfer-quotes`, `/transfers` |
| RFI and limits | `/customers/{re}/rfi`, `/customers/{re}/limit-increase` |
| Partner fees | `/partner-fees` |
| Webhooks | `/webhook-endpoints`, `.../{we}/secret` |
| Upload | `https://api.blindpay.com/v1/upload?instance_id={instance_id}` |

Amounts are always integers in minor units: `100000` is $1,000.00.

## ID prefixes

| Prefix | Resource |
| --- | --- |
| `in_` | Instance |
| `re_` | Customer (formerly receiver) |
| `ba_` | Bank account |
| `bw_` | External blockchain wallet |
| `bl_` | BlindPay-managed wallet |
| `pb_` | Payable (a registered bill) |
| `va_` | Virtual account |
| `qu_` | Payout quote |
| `po_` | Payout |
| `pq_` | Payin quote |
| `pi_` | Payin |
| `tr_` | Transfer |
| `pf_` | Partner fee |
| `we_` | Webhook endpoint |

## Payment methods

Ten bank transfer rails across the US, Brazil, Mexico, Colombia, Argentina, and Europe, plus international SWIFT. Country, currency, and direction (receive/send) per method, including PSE and SEPA's special cases: [Payment methods](references/kb/payment-methods.md). Settlement speed and cut-off times per rail: [Cut-off times](references/kb/cut-off-times.md).

## Chains and tokens

Seven chains: Ethereum, Polygon, Base, Arbitrum, Stellar, Solana, Tron (beta). A production instance sends real USDC/USDT on a mainnet network name (`ethereum`, `polygon`, `base`, `arbitrum`, `stellar`, `solana`, `tron`); a development instance sends **USDB** (BlindPay's test token) on the matching testnet — mismatched combinations are rejected. Which tokens exist on which chain, and the testnet network names, are the enums that gate every quote request: [Supported chains](references/kb/supported-chains.md) is the one place that stays current on them.

## Testing on a development instance

Set the quote's `request_amount` to a sentinel value to force an outcome:

| `request_amount` | Result |
| --- | --- |
| `66600` | `failed` |
| `77700` | `refunded` |
| anything else | `completed` (payins auto-complete about 30 seconds after creation) |

Create a customer with first name `Fail` to simulate a KYC rejection. To fund a test wallet with USDB, create a payin targeting it: on a development instance the payin auto-completes about 30 seconds after creation and delivers USDB into the wallet.

## Customer KYC statuses

Five `kyc_status` values gate what a customer can do: `verifying`, `approved`, `rejected`, `compliance_request`, `approved_rfi`. See [Customers](references/essentials/customers.md) for what triggers each and how long review takes, and [RFI](references/essentials/rfi.md) for the `compliance_request` / `approved_rfi` pair.

## Common errors

- `please_accept_terms_of_service`: the customer must accept the current TOS version
- quote expired: read `expires_at` — never assume a fixed window. Payin, payout, and transfer quotes default to 5 minutes, but OTC (BRL-only) payin quotes expire in 10 seconds and SEPA payout quotes may be shorter; create a new quote once expired. See [Quote expiry windows](references/kb/cut-off-times.md#quote-expiry-windows).
- insufficient balance or allowance: the wallet lacks funds, or the ERC-20 approval is below the quote amount
- KYC not approved: the customer hasn't reached `approved` yet — see Customer KYC statuses above for the full state list and RFI handling
- unsupported token/chain pairing: see [Supported chains](references/kb/supported-chains.md)

## Reference documentation

### Getting started

- [Introduction](references/getting-started/introduction.md) - BlindPay is a global payment API that moves money over bank rails and stablecoins from a single integration.
- [Overview](references/getting-started/overview.md) - Connect to bank rails or stablecoin networks (or both) through one REST API. Issue virtual accounts and move fiat payins and payouts, or hold, send, and receive USDC and USDT across chains.
- [Stablecoins to bank transfer](references/getting-started/quickstart-payout.md) - Send your first off-ramp payout from a BlindPay-managed wallet to a bank account on a development instance, using only the REST API.
- [Bank transfer to stablecoins](references/getting-started/quickstart-payin.md) - Accept a bank transfer and have BlindPay deliver the equivalent stablecoins automatically on a development instance, using only the REST API.
- [SDKs](references/getting-started/sdks.md) - Official BlindPay SDKs for Node.js, Python, Go, PHP, and Swift, plus the OpenAPI spec and REST API reference.
- [Build with AI](references/getting-started/build-with-ai.md) - Connect AI coding agents and AI builders to BlindPay with an MCP server, Agent Skills, and a REST API.

### Essentials

- [Instances](references/essentials/instances.md) - Instances are isolated BlindPay environments, one per stage of your stack, created in the dashboard.
- [Sandbox vs. production](references/essentials/sandbox-vs-production.md) - The exact behavioral differences between development and production instances, plus a checklist for switching over.
- [Billing](references/essentials/billing.md) - How BlindPay charges for virtual accounts, transactions, and partner fees, and when invoices go out
- [Partner fees](references/essentials/partner-fees.md) - Add percentage or flat fees to your customers' transactions and withdraw the accumulated revenue monthly.
- [Customers](references/essentials/customers.md) - People or businesses that send or receive payments and stablecoins through BlindPay.
- [Terms of Service](references/essentials/terms-of-service.md) - A legal agreement your customers must accept before you create them and start KYC.
- [RFI](references/essentials/rfi.md) - Respond programmatically when BlindPay's compliance team needs additional documentation from a customer.
- [Limit increase](references/essentials/limit-increase.md) - Request higher per-transaction, daily, or monthly transfer limits for a customer by submitting a supporting document, and track the request through compliance review.
- [Upload](references/essentials/upload.md) - Generate encrypted file URLs from customer KYC documents and pictures.
- [Analyze document](references/essentials/analyze-document.md) - Read a KYC document with AI and get an approval-rate signal before you submit it.
- [API keys](references/essentials/api-keys.md) - Authenticate all BlindPay API requests with an instance-scoped API key created in the dashboard.
- [Webhooks](references/essentials/webhooks.md) - Receive real-time events for customers, payments, virtual accounts, wallets, and transfers instead of polling the API.
- [Webhooks events](references/essentials/webhooks-events.md) - The full BlindPay webhook event catalog, grouped by domain, with an example payload.
- [Webhook verification](references/essentials/webhooks-verification.md) - Verify the signature on every BlindPay webhook call using the svix-id, svix-timestamp, and svix-signature headers.

### Payouts (money out to a bank account)

- [Payouts](references/payouts/payouts.md) - Execute a payout to a recipient's bank account and track it from processing to completed, failed, or refunded.
- [Bank accounts](references/payouts/bank-accounts.md) - Add recipient bank accounts BlindPay pays out to, across SWIFT, ACH, wire, RTP, Pix, PIX Safe, TED, SPEI, ACH COP, Transfers, and SEPA rails.
- [Payout quotes](references/payouts/payout-quotes.md) - Lock the exchange rate and fee split before executing a payout, and see the exact fee breakdown and recipient amount in the response.
- [Payout with managed wallet](references/payouts/payout-managed-wallet.md) - Fund a payout from a BlindPay-managed wallet with a single REST call, no on-chain approval or signing involved.
- [Payout with EVM](references/payouts/payout-evm.md) - Fund a payout from an external EVM wallet, approve the ERC-20 token transfer, then execute the payout.
- [Payout with Stellar](references/payouts/payout-stellar.md) - Fund a payout from an external Stellar wallet, authorize the payout, sign the XDR transaction, then create the payout.
- [Payout with Solana](references/payouts/payout-solana.md) - Fund a payout from an external Solana wallet, delegate the tokens to BlindPay, then create the payout.
- [Offramp wallets](references/payouts/offramp-wallets.md) - What a BlindPay offramp wallet is, the chains and stablecoins it supports, minimums, and how deposits automatically convert to a fiat payout.

### Payins (money in from a bank account)

- [Payins](references/payins/payins.md) - Create a payin, deliver funds to the destination, and track it through settlement.
- [Payin quotes](references/payins/payin-quotes.md) - Lock the amount, fee split, and destination for a payin before creating it.
- [Blockchain wallets](references/payins/blockchain-wallets.md) - Register an external, customer-controlled wallet address to receive stablecoin payins and send stablecoin payouts.
- [Payin with managed wallet](references/payins/payin-managed-wallet.md) - Accept a fiat payment and have the equivalent stablecoins delivered into a BlindPay-managed wallet, two REST calls with no signing.
- [Payin with blockchain wallet](references/payins/payin-blockchain-wallet.md) - Accept a fiat payment and have the equivalent stablecoins delivered to a wallet your customer controls.

### Payables (paying a registered bill)

- [Payables](references/payables/payables.md) - Register a bill, an invoice, a boleto, or a PIX code, then pay it with the standard payout flow.
- [Boleto and PIX payables](references/payables/payable-boleto-pix.md) - Register a boleto, a utility or tax bill, or a PIX code as a payable, then pay it with the standard payout flow.
- [Invoice payables](references/payables/payable-invoice.md) - Register a supplier invoice paid to a US bank account over ACH or wire, optionally prefilled by reading the PDF with AI.
- [Payable with managed wallet](references/payables/payable-managed-wallet.md) - Register a bill your customer owes, then pay it from a BlindPay-custodied wallet, with no on-chain approval and no wallet prompt.
- [Payable with EVM](references/payables/payable-evm.md) - Register a bill your customer owes, then pay it from an external EVM wallet by quoting it, approving the ERC-20 pull, and executing the payout.

### Virtual accounts

- [Virtual accounts](references/virtual-accounts/virtual-accounts.md) - Issue a virtual account (virtual bank account) that receives USD bank transfers and settles automatically to stablecoins like USDC or USDT in your customer's wallet.
- [Create a virtual account](references/virtual-accounts/virtual-accounts-create.md) - Create a virtual account with the BlindPay API, from customer prerequisites and required compliance fields to the webhooks that confirm approval.

### Wallets, transfers and stablecoins

- [Store](references/wallets/store.md) - Hold a customer's balance in a BlindPay-managed wallet, or register an external wallet the customer already controls.
- [Managed wallets](references/wallets/wallets.md) - Create a BlindPay-managed wallet, check its balance, and use it on payins and payouts.
- [Send](references/wallets/send.md) - Send stablecoins out of a BlindPay-managed wallet to another managed wallet or any external address, using the BlindPay API.
- [Transfer quotes](references/wallets/transfer-quotes.md) - Create a transfer quote to lock in the source wallet, destination address, token, network, and amount before executing a transfer.
- [Transfers](references/wallets/transfers.md) - Execute a stablecoin transfer from a transfer quote and track it through to completion with the BlindPay API.
- [Receive](references/wallets/receive.md) - Receive stablecoins into a BlindPay-managed wallet and track every deposit through the wallet.inbound webhook.

### Knowledge base (compliance, coverage, operations)

- [Cut-off times](references/kb/cut-off-times.md) - ACH, wire, and SWIFT cut-offs and settlement, instant rails, quote expiry windows, onboarding SLAs, and refund timing.
- [Information requests](references/kb/information-requests.md) - A Request for Information (RFI) is how BlindPay compliance asks for missing KYC or KYB details before a customer can be approved.
- [Instance requests](references/kb/instance-requests.md) - How BlindPay compliance asks your team for additional information about your own account, and how to respond before the 27-day window closes.
- [KYB documents](references/kb/kyb-documents.md) - Document requirements for business KYB verification — formation docs, ownership proof, UBO identification, and proof of address.
- [KYC basics](references/kb/kyc-basics.md) - Required KYC verification levels, document-quality standards, and submission guidelines for BlindPay customers.
- [KYC requirements](references/kb/kyc.md) - Verification levels, required fields, statuses, limits, document quality, terms of service, and RFIs for BlindPay customers.
- [NAICS codes](references/kb/naics-codes.md) - Find the NAICS industry code for your business during BlindPay onboarding.
- [Nested payments](references/kb/nested-payments.md) - Nesting is moving money on behalf of a party BlindPay cannot see; this guide explains how to recognize it and stay compliant.
- [On-hold transactions](references/kb/on-hold-transactions.md) - Transactions flagged as suspicious by BlindPay's monitoring system, held pending compliance review or a Request for Information.
- [Payment methods](references/kb/payment-methods.md) - Every bank transfer rail BlindPay supports, by country and currency: ACH, wire, RTP, SWIFT, Pix, PIX Safe, TED, SPEI, PSE, Transfers, and SEPA.
- [Payout descriptor](references/kb/payout-descriptor.md) - How the sender's name appears on a recipient's bank statement varies by payment method and whether Named Account is enabled.
- [POBO and COBO](references/kb/pobo-cobo.md) - How payment on behalf of and collection on behalf of work at BlindPay, what puts your customer's name on a Wire, and where the compliance line sits.
- [Prohibited activities](references/kb/prohibited-activities.md) - High-risk and prohibited business activities at BlindPay, and the disclosure obligations required during onboarding and ongoing monitoring.
- [Proof of address](references/kb/proof-of-address.md) - Accepted proof-of-address documents and submission requirements for business and individual verification.
- [Rejection reasons](references/kb/rejection-reasons.md) - Reason codes and messages returned when an application or document is rejected during verification.
- [Self-custody wallets (Brazil)](references/kb/self-custody-wallets.md) - Brazilian customers must declare whether each external wallet is self-custodied. What changes, who is affected, and how to do it in the dashboard or the API.
- [Source of funds](references/kb/source-of-funds.md) - Documentation required to verify the source of funds and source of wealth for your business.
- [Supported chains](references/kb/supported-chains.md) - Reference for the blockchains, stablecoins, and per-feature chain support across BlindPay payins, payouts, wallets, and transfers.
- [Supported countries](references/kb/supported-countries.md) - Every country BlindPay supports, by tier: standard, high-risk (Enhanced KYC required), and prohibited.
- [SWIFT deliverability](references/kb/swift-deliverability.md) - Requirements for compliance documents and beneficiary address formatting that maximize the chance a SWIFT transfer is delivered.
- [SWIFT statuses](references/kb/swift-statuses.md) - How SWIFT payout compliance documents are tracked through on-hold, review, and approval via the tracking_documents field.
- [Virtual account documents](references/kb/virtual-accounts.md) - Documentation required for the Virtual Account evaluation, including source of funds and source of wealth supporting documents.

### Migration guides (moving from another provider)

- [Migrate from Anchorage Digital to BlindPay](references/migrations/anchorage.md) - Keep Anchorage Digital for custody and move the stablecoin-to-fiat leg to BlindPay: map transfers, withdrawals, and settlement events to BlindPay quotes, payouts, and webhooks.
- [Migrate from BitGo to BlindPay](references/migrations/bitgo.md) - Move the stablecoin-to-fiat leg of a BitGo integration to BlindPay: map wallets, transfers, and settlement webhooks to BlindPay customers, quotes, payouts, and webhook events.
- [Migrate from Bridge to BlindPay](references/migrations/bridge.md) - Move an existing Bridge integration to the BlindPay API: map customers, external accounts, liquidation addresses, and transfers to their BlindPay equivalents.
- [Migrate from Cobo to BlindPay](references/migrations/cobo.md) - Move the stablecoin-to-fiat leg of a Cobo Payments integration to BlindPay: keep Cobo for wallet custody if you want, and replace top-up addresses, order mode, and payout destinations with customers, quotes, and payouts.
- [Migrate from Coinbase CDP to BlindPay](references/migrations/coinbase-cdp.md) - Move the stablecoin-to-fiat offramp leg of a Coinbase Developer Platform integration to BlindPay: add Pix, SPEI, SEPA, and wire payouts while CDP wallets keep custody.
- [Migrate from Conduit to BlindPay](references/migrations/conduit.md) - Move a Conduit cross-border payments integration to BlindPay: counterparties, corridors, and settlement tracking on the quote-and-execute model.
- [Migrate from Crossmint to BlindPay](references/migrations/crossmint.md) - Move the stablecoin-to-fiat leg of a Crossmint integration to BlindPay while Crossmint keeps handling wallets, checkout, or orchestration.
- [Migrate from Dfns to BlindPay](references/migrations/dfns.md) - Keep Dfns for MPC wallet custody and move the stablecoin-to-fiat leg to BlindPay: map transfers and exchange withdrawals to quotes, payouts, and registered external wallets.
- [Migrate from Dynamic to BlindPay](references/migrations/dynamic.md) - Keep Dynamic for embedded and server wallets, move the stablecoin-to-fiat offramp leg to BlindPay: register the Dynamic wallet, quote and execute payouts, and verify webhooks.
- [Migrate from Fern to BlindPay](references/migrations/fern.md) - Move a Fern stablecoin integration to BlindPay: customers, bank accounts, wallets, quote-then-execute payins and payouts, and Svix-signed webhooks.
- [Migrate from Fireblocks to BlindPay](references/migrations/fireblocks.md) - Move the stablecoin-to-fiat payout leg of a Fireblocks integration to BlindPay: re-onboard payees, rebuild the quote-then-payout flow, and port webhooks, while Fireblocks stays as custodian.
- [Migrate from manual payouts to BlindPay](references/migrations/manual-payouts.md) - Turn a spreadsheet-and-bank-portal payout operation into an automated, webhook-driven API flow with an auditable state machine.
- [Migrate from Privy to BlindPay](references/migrations/privy.md) - Keep Privy for embedded and server wallets, move the stablecoin-to-fiat offramp leg to BlindPay: register the wallet, quote and execute payouts, and verify webhooks.
- [Migrate from SWIFT wires to BlindPay](references/migrations/swift-wires.md) - Replace multi-day international wires with same-day stablecoin settlement over local rails, keeping SWIFT as a fallback for corridors BlindPay doesn't cover.
- [Migrate from Turnkey to BlindPay](references/migrations/turnkey.md) - Keep Turnkey for wallets and signing, and move the stablecoin-to-fiat offramp leg of a Turnkey-based product to BlindPay: map payout accounts, quotes, and webhooks.
- [Migrate from Utila to BlindPay](references/migrations/utila.md) - Keep Utila as custodian and signer, and route the stablecoin-to-fiat leg through BlindPay: map vaults, wallets, transactions, and webhooks to their BlindPay equivalents.

### AI tooling integrations

- [Bolt.new + BlindPay](references/integrations/bolt.md) - Add stablecoin payments to a Bolt.new (StackBlitz) app: payout, on-ramp, and virtual-account flows via the BlindPay API and a copy-paste prompt.
- [Claude Code + BlindPay](references/integrations/claude-code.md) - Connect Claude Code to BlindPay via the MCP server and Agent Skills: run payouts, quotes, and virtual accounts from the terminal in natural language.
- [Codex + BlindPay](references/integrations/codex.md) - Connect OpenAI Codex to BlindPay via the MCP server and Agent Skills: run payouts, quotes, and virtual accounts from the terminal or IDE.
- [Cursor + BlindPay](references/integrations/cursor.md) - Connect Cursor to BlindPay via the MCP server and Agent Skills: move money, run payouts, and query corridors in natural language.
- [Hermes Agent + BlindPay](references/integrations/hermes.md) - Connect Hermes Agent to BlindPay with the MCP server and Agent Skills. The Nous Research agent runs stablecoin payouts and quotes via natural language.
- [Lovable + BlindPay](references/integrations/lovable.md) - Add stablecoin payments to your Lovable app: payouts, USDC/USDT on-ramp to fiat, and virtual accounts via the BlindPay API and a copy-paste prompt.
- [OpenClaw + BlindPay](references/integrations/openclaw.md) - Connect OpenClaw to BlindPay with the MCP server and Agent Skills. The autonomous agent runs stablecoin payouts and quotes via natural language.
- [Replit + BlindPay](references/integrations/replit.md) - Add stablecoin payments to a Replit app with Replit Agent: payout, on-ramp, and virtual-account flows via the BlindPay API and Replit Secrets.
- [v0 + BlindPay](references/integrations/v0.md) - Add stablecoin payments to a v0 (Vercel) Next.js app: payout, on-ramp, and virtual-account flows via the BlindPay API and a copy-paste prompt.
