# Terms of Service

A legal agreement your customers must accept before you create them and start KYC.

Source: https://blindpay.com/docs/learn/terms-of-service

BlindPay's terms of service is a legal agreement your customers must accept before you can create them. Acceptance is required for regulatory compliance and lets BlindPay provide services such as issuing blockchain wallets and virtual accounts on their behalf. The terms can only be accepted by a user on the client side at `https://app.blindpay.com`; requests from servers are ignored.

## How it works

When a user accepts the terms, BlindPay redirects them to your `redirect_url` with a `tos_id` query parameter and emits a `tos.accept` webhook. You pass that `tos_id` when [creating a customer](customers.md).

## Prerequisites

**Before you start:**

1. Create an account at https://app.blindpay.com/sign-up
2. Create a development instance (see essentials/instances.md)
3. Create your API key (see essentials/api-keys.md)

## Generate a terms of service URL

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID.

**Note:**

The API only accepts a `uuid` on the `idempotency_key` field. Reusing the same key on a second request is rejected.

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/e/instances/in_000000000000/tos \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "idempotency_key": "<your_uuid>"
  }'
```

The response is a URL with the following query parameters:

```bash [URL example]
https://app.blindpay.com/e/terms-of-service?session_token=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...&idempotency_key=5d8b149e-a55d-4b5b-a8f8-7c4fa315f854&instance_id=in_000000000000&redirect_url=
```

| Param | Required | Example |
| --- | --- | --- |
| `session_token` | Yes | JWT |
| `idempotency_key` | Yes | uuid |
| `instance_id` | Added by BlindPay | `in_000000000000` (used to load your instance logo, name and accent color on the page) |
| `redirect_url` | No | `https://yourapp.com/` |
| `customer_id` | No | `re_000000000000` (required when accepting a new terms of service version) |

We strongly recommend adding a `redirect_url` so the customer lands back in your application after accepting.

The page shows your instance logo and name, and paints the accept button with your accent color, when set. Configure both in the dashboard under Settings, Instance.

## Accept the terms of service

Open the generated URL for the customer to accept on `https://app.blindpay.com`. After acceptance, BlindPay redirects to your `redirect_url` and appends a `tos_id` query parameter. Copy that `tos_id`: you'll pass it as `tos_id` when you [create the customer](customers.md).

![Terms of service acceptance screen showing where to copy the tos_id](https://pub-4fabf5dd55154f19a0384b16f2b816d9.r2.dev/blindpay_tos_acceptance-min.jpg)

You also receive a `tos.accept` webhook event when the terms of service is accepted.

## Accepting a new version

Each `tos_id` is tied to the terms of service version active when it was generated, and can only ever be linked to one customer. If BlindPay updates the terms, calls to the payout quote and payin quote endpoints return an error with message `please_accept_terms_of_service` for customers whose acceptance predates the new version. Generate a new URL, set `customer_id` on it, and have the customer accept again. Once accepted, the quote endpoints stop returning the error.

## Related

- [Customers](customers.md): pass the `tos_id` when creating a customer
- [Instances](instances.md) · [API keys](api-keys.md)
- [KYC](../kb/kyc.md): verification levels and the full onboarding flow
