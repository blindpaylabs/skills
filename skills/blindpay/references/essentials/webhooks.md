# Webhooks

Receive real-time events for customers, payments, virtual accounts, wallets, and transfers instead of polling the API.

Source: https://blindpay.com/docs/learn/webhooks

A webhook is an HTTP callback that BlindPay sends to your server when something changes: a payin completes, a customer is created, a virtual account is approved. Subscribe once and receive every event as it happens, instead of polling the API for status.

Webhooks are configured per [instance](instances.md). You can register up to 25 endpoints on the same instance if you want to split traffic across services.

## Create a webhook

Register your endpoint URL on the instance. The URL must be `https`; local or private addresses are rejected. Pass an empty `events` array to receive every event, or list specific events to subscribe to only those.

**Note:**

Creating, listing, or deleting a webhook endpoint, and fetching its signing secret or portal-access link, all require the `owner`, `admin`, or `developer` role on the instance.

**Note:**

Registering an endpoint is rate-limited to 5 requests per 60 seconds per user. Exceeding it returns a `429` with `webhook_creation_rate_limit_exceeded`; space out the calls if you are provisioning several endpoints in a loop.

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID.

```bash [cURL]
curl --request POST \
  --url https://api.blindpay.com/v1/instances/in_000000000000/webhook-endpoints \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --header 'Content-Type: application/json' \
  --data '{
    "url": "https://example.com/webhook",
    "events": []
  }'
```

The response returns the endpoint ID (`we_...`). To verify the signatures on incoming calls, fetch the endpoint's signing secret:

```bash [cURL]
curl --request GET \
  --url https://api.blindpay.com/v1/instances/in_000000000000/webhook-endpoints/we_000000000000/secret \
  --header 'Authorization: Bearer YOUR_API_KEY'
```

This returns `{ "key": "whsec_..." }`; see [Verification](webhooks-verification.md) for how to use it.

You can also manage endpoints from the [BlindPay dashboard](https://app.blindpay.com), under the instance's **Webhooks** tab.

For quick testing before you have a real handler, point the endpoint at [webhook.cool](https://webhook.cool/): it gives you a temporary URL that logs every incoming request so you can inspect a payload first.

## Open the hosted webhook portal

Instead of building your own UI, you can hand a teammate a link into Svix's hosted webhook portal, where they can inspect endpoints, browse delivery attempts, and replay events, without giving them dashboard access.

```bash [cURL]
curl --request GET \
  --url https://api.blindpay.com/v1/instances/in_000000000000/webhook-endpoints/portal-access \
  --header 'Authorization: Bearer YOUR_API_KEY'
```

The response is `{ "url": "..." }`, a one-time magic link. The link is scoped to the whole instance, not to a single endpoint: anyone who opens it can see and manage every webhook endpoint registered on the instance.

**Abstracted flavor (bank-rails framing, fiat-first API surface)**

### Key events for the Abstracted flavor

Working with the Abstracted flavor, you will listen most often to:

- `virtualAccount.complete`: the account is approved and ready to receive deposits
- `payin.complete`: a deposit arrived and settled
- `payout.complete`: a bank transfer was sent, failed, or was refunded

See [Events](webhooks-events.md) for the full catalog.

**Advanced flavor (stablecoin and blockchain mechanics)**

### Key events for stablecoin

Working with the stablecoin flavor, you will listen most often to:

- `wallet.inbound`: stablecoins were deposited into a managed wallet
- `payin.complete`: fiat was received and stablecoins were delivered
- `payout.complete`: stablecoins were pulled and fiat was sent

See [Events](webhooks-events.md) for the full catalog.

**Note:**

Every webhook call is signed. Verify the `svix-id`, `svix-timestamp`, and `svix-signature` headers before trusting a payload; see Verification for the full process.

## Allowlist the source IPs

If your firewall only accepts traffic from known addresses, allow these. Webhooks come from the same addresses in sandbox and production.

```text [Source IPs]
44.228.126.217
50.112.21.217
52.24.126.164
54.148.139.208
2600:1f24:64:8000::/56
```

These addresses are shared with other senders, so an allowlist alone does not prove a call came from BlindPay. Keep verifying the signature on every call.

## Idempotent requests

Pass an `Idempotency-Key` header on `POST` or `DELETE` calls to this API to make retries safe:

| Situation | What happens |
| --- | --- |
| Same key, identical body, request already finished | The stored response replays verbatim, with an `Idempotency-Replayed: true` response header |
| Same key, a different body | `422` `idempotency_key_payload_mismatch` |
| Same key, the original request is still in flight | `409` `idempotency_key_in_flight` with a `Retry-After` header, unless the original request is older than 5 minutes, in which case it is retaken and re-run |
| Original request failed with a `5xx` | The key is released; retrying with the same key re-runs the operation |

Idempotency keys are not enforced on unauthenticated requests, and are skipped for multipart request bodies. Keys are retained for 24 hours.

## In this section

- [Events](webhooks-events.md): the full event catalog, payload shapes, and when each one fires
- [Verification](webhooks-verification.md): how to verify the signature headers on every call

## Related

- [Instances](instances.md): webhooks are configured per instance
- [API keys](api-keys.md): authenticate the rest of the API alongside your webhook endpoint
- [Partner fees](partner-fees.md): how partner fees are collected and reported on quotes
- [Payin quickstart](../getting-started/quickstart-payin.md): see webhooks in a full payment flow
- [Payout quickstart](../getting-started/quickstart-payout.md): see webhooks in a payout flow
