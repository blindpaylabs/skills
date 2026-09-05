# Transfer quotes

Create a transfer quote to lock in the source wallet, destination address, token, network, and amount before executing a transfer.

Source: https://blindpay.com/docs/transfer-quotes

A transfer quote locks in the details of a stablecoin move before you execute it: the source managed wallet, the destination address, the token, the network, and the amount. Executing the [transfer](transfers.md) simply consumes this quote, it does not take any new parameters of its own.

**Warning:**

`customer_token` must always match `sender_token`, there is no token conversion. `customer_network` must match the source wallet's own network, except when both tokens are `USDC`: a USDC transfer can additionally cross chains using [Circle CCTP v2](#cross-chain-usdc-transfers-circle-cctp-v2).

## How it works

- **Source**: a managed wallet (`wallet_id`, `bl_...`) that holds the stablecoin balance. This is the only supported source, unlike payouts, which can also draw from an external blockchain wallet.
- **Destination**: any blockchain address, `customer_wallet_address`, normally on the same network as the source wallet; a USDC transfer can target a different [CCTP v2 network](#cross-chain-usdc-transfers-circle-cctp-v2) instead. It does not need to belong to a BlindPay customer or wallet, it can be any address the recipient controls.
- **Expiry is 5 minutes**, fixed and not configurable. A transfer quote is meant to be executed immediately after creation, not held for later use: call [create a transfer](transfers.md) right after creating the quote, and request a new one if `expires_at` passes.

**Note:**

`expires_at` is returned in epoch milliseconds.

## Cross-chain USDC transfers (Circle CCTP v2)

When both `sender_token` and `customer_token` are `USDC`, `customer_network` can be a different network than the source wallet's. BlindPay bridges the move using [Circle's Cross-Chain Transfer Protocol v2](https://developers.circle.com/cctp): USDC is burned on the source chain and an equivalent amount is minted on the destination chain once Circle attests the burn.

Supported networks are Ethereum, Polygon, Base, and Arbitrum on a production instance; on a development instance, the matching testnets (Ethereum Sepolia, Polygon Amoy, Base Sepolia, Arbitrum Sepolia), using real testnet USDC rather than USDB, since CCTP can only move native USDC. Both the source and destination network must be on the same side of that mainnet/testnet boundary. Requesting a token other than USDC across networks returns `cross_chain_transfers_only_support_usdc`; requesting a network pair CCTP v2 does not support returns `cctp_route_not_supported`.

The quote itself is unaffected by the bridge: amounts stay 1:1 and BlindPay adds no fee on top, exactly like a same-network transfer. Circle's protocol deducts its own small fee (typically a fraction of a cent up to a few cents, a few basis points of the amount) from the amount minted on the destination chain; this is not reflected in `receiver_amount` or any other quote field. Cross-chain transfers typically settle in 8 to 20 seconds; transfers sourced from Polygon settle in around 8 seconds either way.

Solana, Stellar, Tron, and Tempo are not part of cross-chain transfers yet; same-network moves on those chains are unaffected.

## Prerequisites

**Before you start:**

1. Create an account at https://app.blindpay.com/sign-up
2. Create a development instance (see essentials/instances.md)
3. Create your API key (see essentials/api-keys.md)

You also need a [customer](../kb/kyc.md) whose `kyc_status` is `approved`, `approved_rfi`, or `compliance_request`, and a [managed wallet](wallets.md) holding the stablecoin balance you want to move. This is wider than payins and payouts, which require `approved` or `approved_rfi` only: a customer paused by an open RFI (`compliance_request`) can still move funds already sitting in their managed wallet through a transfer, even though creating a new payin or payout for them stays blocked until the RFI is resolved. Any other `kyc_status` fails with `400` `kyc_not_approved`.

Creating a transfer quote also requires the caller to hold the transaction-create permission on the instance. Dashboard roles that have it: `owner`, `admin`, `finance`, `checker`, and `operations`. A caller without it gets a `403` `user_not_allowed` before `wallet_id` or any other field is checked.

## Request fields

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `wallet_id` | string (`bl_...`) | yes | The managed wallet the funds move out of. Must belong to your instance and not be deleted. |
| `amount_reference` | enum | yes | `sender` or `receiver`. Since sender and receiver amounts are always equal for transfers today, this only affects which field you read the amount from in your own bookkeeping. |
| `request_amount` | integer | yes | Amount in minor units, no floats. `$100.00` sends as `10000`. Minimum is `1`. |
| `sender_token` | enum | yes | `USDC`, `USDT`, or `USDB`. Must be a token allowed on the source wallet's network. |
| `customer_token` | enum | yes | Must equal `sender_token`. |
| `customer_network` | enum | yes | Must equal the source wallet's own network, unless `sender_token` and `customer_token` are both `USDC` and the pair is a [supported CCTP v2 route](#cross-chain-usdc-transfers-circle-cctp-v2). |
| `customer_wallet_address` | string | yes | The destination blockchain address, 32 to 64 characters, validated for the target network (see table below). Can be another wallet you created or any external address. |
| `cover_fees` | boolean | yes | Accepted for API consistency with payin and payout quotes, but fee math is not yet active for transfers: `sender_amount` and `receiver_amount` are always equal to `request_amount`. |
| `partner_fee_id` | string (`pf_...`) | no | Accepted, but partner fee amounts are not yet computed for transfers. |

Address format by network family:

| Network family | Format | Example |
| --- | --- | --- |
| EVM (`ethereum`, `polygon`, `base`, `arbitrum`, `tempo`, and their testnets) | `0x` + 40 hex characters (42 total) | `0xDD6a3aD0949396e57C7738ba8FC1A46A5a1C372` |
| `stellar`, `stellar_testnet` | `G` + 55 base32 characters (56 total) | `GAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA` |
| `solana`, `solana_devnet` | base58, 32 to 44 characters | `11111111111111111111111111111111111111111` |
| `tron` | `T` + 33 base58 characters (34 total) | `TAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA` |

An address that doesn't match the format for `customer_network` fails with `VALIDATION_FAILED` and a message naming the field: `Invalid wallet address for network {network}`.

**Note:**

USDT transfers are currently restricted to the Polygon network only, a tighter limit than payins and payouts allow for USDT.

## Create a transfer quote

**Remember:** replace `YOUR_API_KEY` with your API key, `in_000000000000` with your instance ID, `re_000000000000` with your customer ID.

```bash [cURL]
curl https://api.blindpay.com/v1/instances/in_000000000000/transfer-quotes \
  --request POST \
  --header 'Content-Type: application/json' \
  --header 'Authorization: Bearer YOUR_API_KEY' \
  --data '{
  "wallet_id": "bl_000000000000",
  "amount_reference": "sender",
  "cover_fees": true,
  "request_amount": 1000,
  "sender_token": "USDC",
  "customer_wallet_address": "0xDD6a3aD0949396e57C7738ba8FC1A46A5a1C372",
  "customer_token": "USDC",
  "customer_network": "polygon",
  "partner_fee_id": "pf_000000000000"
}'
```

```js [index.js]
const response = await fetch(
  'https://api.blindpay.com/v1/instances/in_000000000000/transfer-quotes',
  {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: 'Bearer YOUR_API_KEY',
    },
    body: JSON.stringify({
      wallet_id: 'bl_000000000000',
      amount_reference: 'sender',
      cover_fees: true,
      request_amount: 1000,
      sender_token: 'USDC',
      customer_wallet_address: '0xDD6a3aD0949396e57C7738ba8FC1A46A5a1C372',
      customer_token: 'USDC',
      customer_network: 'polygon',
      partner_fee_id: 'pf_000000000000',
    }),
  },
)

const quote = await response.json()
```

## Response

```json
{
  "id": "qu_000000000000",
  "expires_at": 1712958191000,
  "commercial_quotation": 100,
  "blindpay_quotation": 100,
  "receiver_amount": 1000,
  "sender_amount": 1000,
  "partner_fee_amount": 0,
  "flat_fee": 0
}
```

| Field | Type | Notes |
| --- | --- | --- |
| `id` | string (`qu_...`) | Pass this as `transfer_quote_id` when you create the transfer. |
| `expires_at` | number | Epoch milliseconds. Execute the transfer before this passes. |
| `commercial_quotation` | number | Rate preview. Currently fixed at `100` for transfers since there is no FX conversion, even for a cross-chain USDC move. |
| `blindpay_quotation` | number | Same as `commercial_quotation` for transfers today. |
| `receiver_amount` | number | Equal to `request_amount`. |
| `sender_amount` | number | Equal to `request_amount`. |
| `partner_fee_amount` | number | Always `0` for transfers today, even if `partner_fee_id` was set. |
| `flat_fee` | number | Always `0` for transfers today. |

**Note:**

The `qu_` prefix is shared across payin quotes, payout quotes, and transfer quotes, you can't tell which kind of quote an ID refers to just by looking at it.

## Validation order

The API checks, in this order: the caller has permission to create transactions, the wallet exists and belongs to your instance. If `customer_network` differs from the source wallet's network, the API requires both `sender_token` and `customer_token` to be `USDC` (`cross_chain_transfers_only_support_usdc` otherwise) and the network pair to be a [supported CCTP v2 route](#cross-chain-usdc-transfers-circle-cctp-v2) (`cctp_route_not_supported` otherwise). If `customer_network` matches the source wallet's network, the API instead checks that the token and network are allowed for your instance type, that USDT is only used on Polygon, and that `sender_token` matches `customer_token`. Either path finishes with a check that the receiving customer's `kyc_status` is one of `approved`, `approved_rfi`, or `compliance_request`. The first failing check is the one returned.

## Testing

There is no dedicated test-amount sentinel for transfer quotes or transfers (unlike payins and payouts, which force outcomes at `$666.00` and `$777.00`). On development instances, transfer quotes go through the same token and network rules as production, restricted to the development tokens and testnets described in [supported chains](../kb/supported-chains.md). Cross-chain USDC quotes are the exception: on a development instance they use real testnet USDC, not USDB, on the CCTP v2 testnets (Ethereum Sepolia, Polygon Amoy, Base Sepolia, Arbitrum Sepolia), since Circle's protocol can only move native USDC.

## Related

- [Send](send.md): the concept overview and scope for transfers
- [Transfers](transfers.md): execute a transfer against this quote
- [Managed wallets](wallets.md): create and fund the source wallet
- [Payout quotes](../payouts/payout-quotes.md): compare against the quote used for off-ramp payouts
- [Supported chains](../kb/supported-chains.md): token and network support matrix
