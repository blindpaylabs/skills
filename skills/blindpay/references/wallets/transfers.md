# Transfers

Execute a stablecoin transfer from a transfer quote and track it through to completion with the BlindPay API.

Source: https://blindpay.com/docs/transfers

A transfer is the object that actually moves stablecoins: it consumes a [transfer quote](transfer-quotes.md) and sends the funds from a managed wallet to the destination address locked in on that quote. The transfer itself takes a single field; the amount, token, network, and destination were already set when you created the quote.

**Warning:**

USDC moves can now cross chains between Ethereum, Polygon, Base, and Arbitrum using [Circle CCTP v2](transfer-quotes.md#cross-chain-usdc-transfers-circle-cctp-v2); every other token still requires the destination network to match the source wallet's exactly.

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

Replace `qu_000000000000` with the transfer quote ID you created previously. The call is the same whether the quote is a same-network transfer, a cross-chain USDC transfer, or a USDC and EURC conversion; see [examples by transfer type](transfer-quotes.md#examples-by-transfer-type) for the quote that sets each one up.

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

`tracking_partner_fee` is also present: `null` when the quote has no partner fee, otherwise an object that opens at `on_hold`. See [Partner fee collection](#partner-fee-collection).

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
| `tracking_paymaster` | The collection of `sender_token` from the source wallet in a [USDC and EURC conversion](#usdc-and-eurc-conversion); stays `on_hold` for every other transfer | `step`, `transaction_hash`, `gas_fee`, `completed_at`, `error_message` |
| `tracking_bridge_swap` | The burn leg of a cross-chain USDC transfer, or the conversion leg of a [USDC and EURC conversion](#usdc-and-eurc-conversion); stays `on_hold` on any other same-network transfer | `step`, `transaction_hash`, `gas_fee`, `completed_at`, `error_message` |
| `tracking_complete` | The send (same-network) or mint (cross-chain) leg | `step`, `transaction_hash`, `gas_fee`, `completed_at`, `error_message` |

`transaction_hash`, `gas_fee`, `completed_at`, and `error_message` are all `null` until that step produces a value; `error_message` is set only when the step fails. `blockchain_screening` and `risk_score` are BlindPay's own compliance scoring for the transfer, not something you set.

## USDC and EURC conversion

When the quote converts between `USDC` and `EURC` on Base (`USDB` and `EURB` on a development instance), executing the transfer first books the conversion at the current rate, then creates the transfer:

- If the rate moved more than 0.5% against the customer since the quote, the call fails with `409` `fx_rate_moved` and no transfer is created. Create a new quote.
- The source wallet must hold `sender_amount` before the conversion is booked, otherwise the call fails with `400` `not_enough_balance`.
- If the provider refuses the conversion, the call fails with `400` (`fx_trade_rejected`, or `fx_amount_below_minimum` under the 10 USDC minimum) and no transfer is created.
- If the provider does not confirm in time, the call fails with `502` `fx_trade_unconfirmed` and no funds move; BlindPay reconciles the conversion on its side.
- A quote can only be executed once: a second attempt fails with `400` `fx_quote_already_used`.

Once created, the transfer moves through three legs:

1. `tracking_paymaster`: `sender_amount` minus any partner fee leaves the source wallet in `sender_token`. `transfer.update` fires.
2. `tracking_bridge_swap`: the funds are converted into `customer_token`. `transfer.update` fires once the conversion settles; its `transaction_hash` is the collection transaction from step 1.
3. `tracking_complete`: exactly `receiver_amount` in `customer_token` is sent to `customer_wallet_address`. `status` becomes `completed` and `transfer.complete` fires.

If a leg after the conversion is booked cannot finish, the transfer stays `processing` while BlindPay resolves it; it only moves to `failed` when collecting from the source wallet fails, in which case nothing reaches the destination.

## Partner fee collection

When the [transfer quote](transfer-quotes.md#partner-fees) carries a partner fee, the transfer moves funds in two legs out of the source wallet:

1. `receiver_amount` goes to `customer_wallet_address`, tracked on `tracking_complete` (and `tracking_bridge_swap` for a cross-chain transfer). `status` follows this leg: the transfer is `completed` and `transfer.complete` fires once it confirms.
2. After that, `partner_fee_amount` goes to BlindPay, tracked on `tracking_partner_fee`, on the source wallet's network.

`tracking_partner_fee` has the same fields as the other blockchain tracking objects (`step`, `transaction_hash`, `gas_fee`, `completed_at`, `error_message`) and moves from `on_hold` to `processing` to `completed`, usually a few seconds after the transfer completes. No separate webhook fires for this leg: [retrieve the transfer](#retrieve-a-transfer) to read it. A fee is only counted towards your monthly [partner fee](../essentials/partner-fees.md) payout once `tracking_partner_fee.step` is `completed`.

### Balance check

The source wallet must hold the full `sender_amount` (transfer plus fee) before anything moves, so a transfer never reaches its destination without its fee:

- For a wallet without [yield](https://blindpay.com/docs/yield), executing the transfer fails with `400` `not_enough_balance` and no transfer is created.
- For a wallet with yield enabled, the check runs after BlindPay pulls the needed funds back from the yield position. If the balance is still short, the transfer is created and then moves to `failed` with `tracking_complete.error_message` set to `not_enough_balance`, before any funds are sent. `transfer.update` fires.

Transfers without a partner fee are not affected.

## Testing

Transfers do not use the `66600`/`77700` sentinel amounts that payins and payouts support. On a development instance, a same-network transfer executes against the sandbox `USDB` token on the corresponding testnet and confirms once the network finalizes the send. A cross-chain USDC transfer is the exception: it uses real testnet USDC, not USDB, since [Circle CCTP v2](transfer-quotes.md#cross-chain-usdc-transfers-circle-cctp-v2) can only move native USDC. A [USDC and EURC conversion](#usdc-and-eurc-conversion) runs between USDB and EURB on Base Sepolia and goes through the same three legs and webhooks as on production, so you can build against it end to end.

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
