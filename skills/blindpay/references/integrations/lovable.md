# Lovable + BlindPay

Add stablecoin payments to your Lovable app: payouts, USDC/USDT on-ramp to fiat, and virtual accounts via the BlindPay API and a copy-paste prompt.

Source: https://blindpay.com/docs/integrations/lovable

[Lovable](https://lovable.dev) builds full-stack apps from natural language. This guide adds **stablecoin payments**: cross-border payouts, on-ramps, and virtual accounts: to a Lovable project using the BlindPay REST API.

## Copy-paste prompt

Paste this into Lovable to scaffold a stablecoin payout flow:

```text [Lovable prompt]
Add stablecoin payments to this app using the BlindPay API (https://api.blindpay.com).

Requirements:
- Create a backend function that calls BlindPay to:
  1. Create a payout quote (POST /v1/instances/{instance_id}/payouts/evm/quote)
  2. Execute a payout (POST /v1/instances/{instance_id}/payouts/evm)
- Read BLINDPAY_API_KEY and BLINDPAY_INSTANCE_ID from secrets.
- Authenticate with: Authorization: Bearer ${BLINDPAY_API_KEY}
- Build a form where a user enters an amount in USDC and a destination
  (bank account / blockchain wallet), shows the live quote, and submits the payout.
- Never expose the API key in client code: all BlindPay calls go through the backend.

Docs: https://blindpay.com/docs/introduction
```

## Setup

### Get your credentials

Create an account and a development instance, then copy your API key and instance ID from the [BlindPay dashboard](https://app.blindpay.com/sign-up/).

### Add secrets in Lovable

In your Lovable project settings (or connected Supabase), add:

```bash [Secrets]
BLINDPAY_API_KEY=your-api-key
BLINDPAY_INSTANCE_ID=your-instance-id
```

### Paste the prompt

Paste the prompt above. Lovable generates a backend function plus UI. Review the generated calls against the [API reference](https://api.blindpay.com/reference).

## Example backend call

```ts [Backend function]
const res = await fetch(
  `https://api.blindpay.com/v1/instances/${instanceId}/payouts/evm/quote`,
  {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.BLINDPAY_API_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ currency_type: 'sender', request_amount: 1000 }),
  },
)
const quote = await res.json()
```

**Note:**

Keep the secret API key server-side. In Lovable that means a backend/edge function: never a client component.

## Next steps

- [Quick start: stablecoin to fiat](../getting-started/quickstart-payout.md)
- [All AI builder integrations](https://blindpay.com/docs/integrations)
- [BlindPay for AI agents](../getting-started/build-with-ai.md)
