# Replit + BlindPay

Add stablecoin payments to a Replit app with Replit Agent: payout, on-ramp, and virtual-account flows via the BlindPay API and Replit Secrets.

Source: https://blindpay.com/docs/integrations/replit

[Replit](https://replit.com) and Replit Agent build and host full-stack apps. This guide adds **stablecoin payments**: payouts, on-ramps, and virtual accounts: with the BlindPay REST API and Replit Secrets.

## Copy-paste prompt

Give this to Replit Agent:

```text [Replit Agent prompt]
Integrate stablecoin payments using the BlindPay API (https://api.blindpay.com).

- Add a backend route that creates a payout quote and executes a payout:
  POST /v1/instances/${BLINDPAY_INSTANCE_ID}/payouts/evm/quote and /payouts/evm.
- Authenticate with Authorization: Bearer ${BLINDPAY_API_KEY}.
- Read BLINDPAY_API_KEY and BLINDPAY_INSTANCE_ID from Replit Secrets (environment).
- Build a UI: enter a USDC amount and destination, show the live quote, send the payout.
- Keep the secret key server-side only.

Docs: https://blindpay.com/docs/getting-started/overview
```

## Setup

### Get your credentials

Copy your API key and instance ID from the [BlindPay dashboard](https://app.blindpay.com/sign-up/).

### Add Replit Secrets

Open **Tools → Secrets** and add:

```bash [Replit Secrets]
BLINDPAY_API_KEY=your-api-key
BLINDPAY_INSTANCE_ID=your-instance-id
```

### Prompt the agent

Paste the prompt into Replit Agent. Review the generated route against the [API reference](https://api.blindpay.com/reference).

## Example server call

```ts [server]
const res = await fetch(
  `https://api.blindpay.com/v1/instances/${process.env.BLINDPAY_INSTANCE_ID}/payouts/evm/quote`,
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

Always read the key from Replit Secrets in server code. Never commit it or expose it in the client.

## Next steps

- [Quick start: stablecoin to fiat](../getting-started/quickstart-payout.md)
- [All AI builder integrations](https://blindpay.com/docs/integrations)
- [BlindPay for AI agents](../getting-started/build-with-ai.md)
