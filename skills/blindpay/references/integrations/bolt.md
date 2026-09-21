# Bolt.new + BlindPay

Add stablecoin payments to a Bolt.new (StackBlitz) app: payout, on-ramp, and virtual-account flows via the BlindPay API and a copy-paste prompt.

Source: https://blindpay.com/docs/integrations/bolt

[Bolt.new](https://bolt.new) by StackBlitz builds and runs full-stack apps in the browser from a prompt. This guide adds **stablecoin payments**: payouts, on-ramps, and virtual accounts: using the BlindPay REST API.

## Copy-paste prompt

```text [Bolt.new prompt]
Add stablecoin payments to this app using the BlindPay API (https://api.blindpay.com).

- Create a server route that calls BlindPay to create a payout quote and execute
  a payout: POST /v1/instances/${BLINDPAY_INSTANCE_ID}/payouts/evm/quote and
  POST /v1/instances/${BLINDPAY_INSTANCE_ID}/payouts/evm.
- Authenticate with Authorization: Bearer ${BLINDPAY_API_KEY}.
- Read BLINDPAY_API_KEY and BLINDPAY_INSTANCE_ID from environment variables.
- Build a UI: enter a USDC amount and destination, fetch a live quote, send the payout.
- Keep the secret key on the server, never in client code.

Docs: https://blindpay.com/docs/getting-started/overview
```

## Setup

### Get your credentials

Copy your API key and instance ID from the [BlindPay dashboard](https://app.blindpay.com/sign-up/).

### Add environment variables

```bash [.env]
BLINDPAY_API_KEY=your-api-key
BLINDPAY_INSTANCE_ID=your-instance-id
```

### Paste the prompt

Paste the prompt into Bolt.new and run. Check the generated requests against the [API reference](https://api.blindpay.com/reference).

## Example server call

```ts [server route]
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

Bolt apps run client-side in StackBlitz: route BlindPay calls through a server/edge function so the secret key is never exposed.

## Next steps

- [Quick start: stablecoin to fiat](../getting-started/quickstart-payout.md)
- [All AI builder integrations](https://blindpay.com/docs/integrations)
- [BlindPay for AI agents](../getting-started/build-with-ai.md)
