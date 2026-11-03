# Receive

Receive stablecoins into a BlindPay-managed wallet and track every deposit through the wallet.inbound webhook.

Source: https://blindpay.com/docs/receive

Receive covers stablecoins arriving in a [managed wallet](wallets.md). There is nothing to call: share the wallet's `address` with whoever is sending, and any transfer that arrives on the matching network credits the wallet. Your integration work is on the webhook side, reacting when funds land.

If you're looking for fiat coming in (a bank transfer converted to stablecoins), that's a [payin](../payins/payins.md).

## How it works

1. Create a [managed wallet](wallets.md) for the customer and share its `address`.
2. The sender transfers stablecoins to that address from any wallet or exchange, on the wallet's network.
3. The deposit credits the wallet balance automatically; there is no approval or signature step on the receiving end.
4. BlindPay fires a `wallet.inbound` webhook so your system can react in real time.

**Warning:**

The deposit must arrive on the same network the wallet was created on. Funds sent on the wrong network are not credited and cannot be recovered by BlindPay.

## The wallet.inbound webhook

Every time stablecoins land in a managed wallet, `wallet.inbound` fires with this payload:

```json [wallet.inbound]
{
  "id": "bl_000000000000",
  "address": "0x1234567890abcdef1234567890abcdef12345678",
  "network": "base",
  "token": {
    "address": "0x1234567890abcdef1234567890abcdef12345678",
    "id": "00000000-0000-0000-0000-000000000000",
    "symbol": "USDC",
    "amount": 100
  },
  "deposit_id": "wd_000000000000",
  "transaction_hash": "0x8f3e1b..."
}
```

`wallet.inbound` fires for USDC, USDT, and EURC deposits, plus the test tokens USDB and EURB on development instances. See [webhooks](../essentials/webhooks.md) for endpoint setup and signature verification.

**Warning:**

`wallet.inbound` reports `amount` scaled by 100 (so `100` means $1.00), while the wallet balance endpoint reports the raw amount. Don't assume the two use the same unit.

## Check the balance

Webhooks are the push signal; the balance endpoint is the pull:

```bash [cURL]
curl --request GET \
  --url https://api.blindpay.com/v1/instances/in_000000000000/customers/re_000000000000/wallets/bl_000000000000/balance \
  --header 'Authorization: Bearer YOUR_API_KEY'
```

## Related

- [Managed wallets](wallets.md): create the wallet that receives the deposits
- [Payins](../payins/payins.md): receive fiat and have it delivered as stablecoins instead
- [Send](send.md): move the received stablecoins to another wallet
- [Payouts](../payouts/payouts.md): convert the balance to fiat in a bank account
- [Webhooks](../essentials/webhooks.md): endpoint setup and signature verification
