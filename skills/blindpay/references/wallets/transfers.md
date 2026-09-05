# Transfers

Execute a stablecoin transfer from a transfer quote and track it through to completion with the BlindPay API.

Source: https://blindpay.com/docs/transfers

A transfer is the object that actually moves stablecoins: it consumes a [transfer quote](transfer-quotes.md) and sends the funds from a managed wallet to the destination address locked in on that quote. The transfer itself takes a single field; the amount, token, network, and destination were already set when you created the quote.

**Warning:**

Transfers are gated by `subscription_features.wallets_and_transfers` on your instance. Contact BlindPay to enable it. Calling any transfer endpoint (create, retrieve, or list) without it enabled returns a 400 `wallets_and_transfers_not_enabled` error. USDC moves can now cross chains between Ethereum, Polygon, Base, and Arbitrum using [Circle CCTP v2](transfer-quotes.md#cross-chain-usdc-transfers-circle-cctp-v2); every other token still requires the destination network to match the source wallet's exactly.

## How it works

```
transfer quote (qu_...) -> execute the transfer -> stablecoins sent from the managed wallet -> destination confirms receipt
```

A transfer quote expires 5 minutes after creation, so execute the transfer before then. Executing against an expired quote returns a 400 `quote_expired` error; create a fresh transfer quote and execute against that instead. Once created, a transfer cannot be canceled.

**Before you start:**

1. Create an account at https://app.blindpay.com/sign-up
2. Create a development instance (see essentials/instances.md)
3. Create your API key (see essentials/api-keys.md)

You also need a [managed wallet](wallets.md) (`bl_...`) and a [transfer quote](transfer-quotes.md) (`qu_...`).

## Execute a transfer

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID.

Replace `qu_000000000000` with the transfer quote ID you created previously.

Pass an `Idempotency-Key` header (any string, up to 255 characters) to safely retry this call: replaying the same key with an identical body returns the original transfer instead of starting a second one. The same key with a different body is rejected. Keys are retained for 24 hours.

```bash [cURL]
curl https://api.blindpay.com/v1/instances/in_000000000000/transfers \
  --request POST \
  --header 'Content-Type: application/json' \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --data '{
  "transfer_quote_id": "qu_000000000000"
}'
```

### Response

```json
{
  "id": "tr_000000000000",
  "status": "processing",
  "tracking_transaction_monitoring": { "step": "processing", "blockchain_screening": 0, "risk_score": 0, "completed_at": null },
  "tracking_paymaster": { "step": "on_hold", "transaction_hash": null, "gas_fee": null, "completed_at": null, "error_message": null },
  "tracking_bridge_swap": { "step": "on_hold", "transaction_hash": null, "gas_fee": null, "completed_at": null, "error_message": null },
  "tracking_complete": { "step": "on_hold", "transaction_hash": null, "gas_fee": null, "completed_at": null, "error_message": null }
}
```

`tracking_bridge_swap` opens at `processing` instead of `on_hold` when the transfer is cross-chain, and `tracking_complete` opens at `processing` with `transaction_hash` already set when the source wallet is on a Tempo-managed network. See [Tracking objects](#tracking-objects) below.

## Status lifecycle

| `status` | Meaning | Terminal? |
| --- | --- | --- |
| `processing` | The stablecoin send has been submitted and is waiting for confirmation | no |
| `completed` | The transfer confirmed and the destination received the stablecoins | yes |
| `failed` | The send did not go through | yes |
| `refunded` | The stablecoins were returned to the source wallet instead of reaching the destination | yes |

**Note:**

Check `tracking_complete` alongside `status` when building a detailed view: it carries additional detail about the send once the transfer confirms.

**Note:**

For a cross-chain USDC transfer, `tracking_bridge_swap` tracks the burn on the source chain and `tracking_complete` tracks the mint on the destination chain. A same-network transfer leaves `tracking_bridge_swap` untouched, but still resolves both `tracking_transaction_monitoring` (BlindPay's screening) and `tracking_complete` (the send) once the transfer finishes. On failure, `tracking_complete.step` falls back to `on_hold` rather than advancing to `completed`, and `error_message` is set.

## Tracking objects

Every transfer carries 4 tracking objects, each moving through the same `step` values:

| `step` | Meaning |
| --- | --- |
| `processing` | This step is in flight |
| `on_hold` | Not reached yet, or paused pending an earlier step |
| `pending_review` | Held for manual compliance review |
| `pending_refund_review` | Held for manual review before a refund |
| `completed` | This step finished |

| Object | Tracks | Fields |
| --- | --- | --- |
| `tracking_transaction_monitoring` | BlindPay's compliance screening of the transfer | `step`, `blockchain_screening` (`0`-`100` or `null`), `risk_score` (`0`-`100` or `null`), `completed_at` |
| `tracking_paymaster` | Reserved for the paymaster leg; stays `on_hold` for every transfer today | `step`, `transaction_hash`, `gas_fee`, `completed_at`, `error_message` |
| `tracking_bridge_swap` | The burn leg of a cross-chain USDC transfer; stays `on_hold` on a same-network transfer | `step`, `transaction_hash`, `gas_fee`, `completed_at`, `error_message` |
| `tracking_complete` | The send (same-network) or mint (cross-chain) leg | `step`, `transaction_hash`, `gas_fee`, `completed_at`, `error_message` |

`transaction_hash`, `gas_fee`, `completed_at`, and `error_message` are all `null` until that step produces a value; `error_message` is set only when the step fails. `blockchain_screening` and `risk_score` are BlindPay's own compliance scoring for the transfer, not something you set.

## Testing

Transfers do not use the `66600`/`77700` sentinel amounts that payins and payouts support. On a development instance, a same-network transfer executes against the sandbox `USDB` token on the corresponding testnet and confirms once the network finalizes the send. A cross-chain USDC transfer is the exception: it uses real testnet USDC, not USDB, since [Circle CCTP v2](transfer-quotes.md#cross-chain-usdc-transfers-circle-cctp-v2) can only move native USDC.

## Retrieve a transfer

```bash [cURL]
curl https://api.blindpay.com/v1/instances/in_000000000000/transfers/tr_000000000000 \
  --header 'Authorization: Bearer YOUR_API_KEY'
```

This returns the full transfer, unlike the response you get from executing it. On top of `id`, `status`, and the tracking objects, you get the sender and receiver amount, token, network, and destination address carried over from the transfer quote, the receiving customer's `image_url`, `first_name`, `last_name`, and `legal_name`, and the source wallet's `id`, `external_id`, `address`, and `network`.

## List transfers

```bash [cURL]
curl https://api.blindpay.com/v1/instances/in_000000000000/transfers \
  --header 'Authorization: Bearer YOUR_API_KEY'
```

Returns transfers newest first, wrapped in the standard [pagination](https://blindpay.com/docs/learn/pagination) envelope. Pass `limit`, `starting_after`, or `ending_before` for cursor pagination.

| Param | Notes |
| --- | --- |
| `limit` | One of `10`, `50`, `100`, `200`, `500`, `1000`. Defaults to `10`. |
| `starting_after` | A transfer id. Returns the page of transfers created before that transfer. |
| `ending_before` | A transfer id. Returns the page of transfers created after that transfer. |

The response wraps the array in a `pagination` object:

| Field | Notes |
| --- | --- |
| `has_more` | `true` if more transfers exist past this page. |
| `next_page` | The id of the last transfer in this page, or `null` when `has_more` is `false`. Pass it as `starting_after` to fetch the next page. |
| `prev_page` | Echoes back whichever of `starting_after`/`ending_before` you passed, or `null` if neither was passed. |

## Track a transfer

BlindPay also exposes an unauthenticated tracking endpoint for building a public status page or printed receipt for the recipient:

```bash [cURL]
curl https://api.blindpay.com/v1/e/transfers/tr_000000000000
```

No API key and no `instance_id` are needed, the transfer `id` alone is the lookup key. The response is the same transfer object [retrieve a transfer](#retrieve-a-transfer) returns, including the receiving customer's name and the source wallet's address: nothing on the transfer is stripped for this endpoint, so treat the `id` itself as the access control and don't leak it anywhere you wouldn't share the transfer's details.

## Webhooks

| Event | Fires when |
| --- | --- |
| `transfer.new` | The transfer is created and the stablecoin send has been submitted |
| `transfer.update` | The transfer advances through an intermediate step (for example, a cross-chain burn confirmation), or fails to confirm |
| `transfer.complete` | The transfer confirms and the destination has received the stablecoins |

Check `status` on the `transfer.update` payload: it stays `processing` for an intermediate step, or moves to `failed`/`refunded` when the send does not go through.

See [webhooks](../essentials/webhooks.md) for signature verification and full payload details.

## Related

- [Transfer quotes](transfer-quotes.md): lock in the amount, token, network, and destination before executing a transfer
- [Send](send.md): the transfer concept and how it fits alongside payins and payouts
- [Managed wallets](wallets.md): create and fund the source wallet for a transfer
- [Blockchain wallets](../payins/blockchain-wallets.md): register an external destination address
