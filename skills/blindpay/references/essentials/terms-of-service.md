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

## Passthrough Terms of Service

Passthrough Terms of Service lets you collect the acceptance inside your own onboarding flow instead of sending customers to the hosted page. You present the BlindPay terms on your page, and your server submits the acceptance to BlindPay and receives a `tos_id` to use when you [create the customer](customers.md).

**Note:**

Passthrough Terms of Service is enabled per instance by BlindPay. Contact support to request it. The hosted Terms of Service page keeps working on every instance.

### Set up your instance

#### Add your Terms of Service link and screenshot

In the dashboard, open **Settings**, **Instance**, then the **Passthrough Terms of Service** section. Add the link to the page where your customers see the BlindPay terms, and upload a screenshot of how they are displayed. Through the API, set `partner_tos_screenshot` to a `file_url` returned by [upload](upload.md) with bucket `documents`. Update both whenever the way you present the terms changes.

#### Accept the partner agreement

Read the [Passthrough Terms of Service Partner Agreement](https://blindpay.com/passthrough-tos-partner-agreement) and accept it in the same section. It sets out how you present the terms, the acceptance data you submit, how long you keep records, audits, and the remedies if the data is not genuine. BlindPay records the date, IP address, browser and user of the acceptance.

#### Collect the acceptance data on your page

When the customer takes an explicit action to accept the terms, such as ticking an unchecked checkbox, capture on your side:

- the public IP address of the customer device, as seen by your edge or load balancer
- the complete `User-Agent` header sent by the customer browser

### Submit the acceptance

Call the endpoint from your server right after the customer accepts. BlindPay records the acceptance time when it receives the request.

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/tos/passthrough \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "idempotency_key": "<your_uuid>",
    "ip_address": "191.34.12.8",
    "user_agent": "Mozilla/5.0 (iPhone; CPU iPhone OS 17_5 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/17.5 Mobile/15E148 Safari/604.1"
  }'
```

```ts [Node.js]
const res = await fetch('https://api.blindpay.com/v1/instances/in_000000000000/tos/passthrough', {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${process.env.BLINDPAY_API_KEY}`,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    idempotency_key: crypto.randomUUID(),
    ip_address: customerIp,
    user_agent: request.headers['user-agent'],
  }),
})

const { tos_id } = await res.json()
```

| Field | Required | Description |
| --- | --- | --- |
| `idempotency_key` | Yes | A `uuid`. Reusing the same key on a second request is rejected. |
| `ip_address` | Yes | Public IPv4 or IPv6 address of the customer device. Not the IP address of your servers. |
| `user_agent` | Yes | The complete, unmodified `User-Agent` header sent by the customer browser. |
| `customer_id` | No | An existing customer of this instance to link the acceptance to, for example when they must accept a new terms of service version. |

The response has the same shape as the hosted page acceptance, and BlindPay also emits a `tos.accept` webhook:

```json
{
  "tos_id": "to_000000000000",
  "idempotency_key": "123e4567-e89b-12d3-a456-426614174000",
  "customer_id": null,
  "version": "1.0.1"
}
```

Pass `tos_id` when you [create the customer](customers.md). Each `tos_id` can only be linked to one customer, and only customers of the same instance can use it.

### Validation

BlindPay checks that every submission looks like it came from the customer browser. A request that fails returns `400` with code `TOS_PASSTHROUGH_ATTESTATION_INVALID` and lists every problem in `errors`:

| Rejected | Reason in `errors` |
| --- | --- |
| Private, reserved or test IP addresses | `ip_address must be the public IP address of the customer device` |
| The IP address of the server calling the API | `ip_address is the IP address of the server calling the API, not of the customer device` |
| Non-browser agents such as `curl`, HTTP libraries, headless browsers or bots | `user_agent must be the User-Agent header sent by the browser of the customer` |
| IP addresses of data centers or hosting providers (production instances) | `ip_address belongs to a data center or hosting provider, not to the customer device` |
| Tor exit nodes and proxies (production instances) | `ip_address is a Tor exit node or a proxy, not the customer device` |

Other errors you may see:

| Code | When |
| --- | --- |
| `TOS_PASSTHROUGH_NOT_ENABLED` | Passthrough Terms of Service is not enabled for the instance. |
| `TOS_PASSTHROUGH_SETUP_INCOMPLETE` | The link, the screenshot or the partner agreement is missing in the instance settings. |
| `CUSTOMERS_NOT_FOUND` | `customer_id` does not belong to the instance. |

On production instances BlindPay also checks the IP address with its fraud provider and records its country, so customers from prohibited countries are rejected when you create them. BlindPay monitors submissions over time, for example the same IP address reused across many customers or acceptances from VPNs, and may flag an acceptance for review or require a new acceptance through the hosted page, as set out in the [partner agreement](https://blindpay.com/passthrough-tos-partner-agreement).

## Related

- [Customers](customers.md): pass the `tos_id` when creating a customer
- [Instances](instances.md) · [API keys](api-keys.md)
- [KYC](../kb/kyc.md): verification levels and the full onboarding flow
